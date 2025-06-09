# PostgreSQL Memory Contexts

**Source Headers:**
*   `src/include/utils/memutils.h` (High-level API, standard contexts)
*   `src/include/nodes/memnodes.h` (Core `MemoryContextData` structure, method definitions)
*   `src/include/utils/palloc.h` (Allocation functions like `palloc`, `MemoryContextSwitchTo`)

**Implementation Files:** (Primarily)
*   `src/backend/utils/mmgr/mcxt.c` (Context-independent operations)
*   `src/backend/utils/mmgr/aset.c` (`AllocSetContext` implementation)
*   `src/backend/utils/mmgr/slab.c` (`SlabContext` implementation)
*   `src/backend/utils/mmgr/generation.c` (`GenerationContext` implementation)
*   `src/backend/utils/mmgr/bump.c` (`BumpContext` implementation)

## Overview

PostgreSQL's memory context system is a hierarchical memory management mechanism designed to prevent memory leaks and provide fine-grained control over memory allocation lifetimes. Instead of direct calls to `malloc` and `free` for most backend operations, PostgreSQL uses `palloc` (PostgreSQL alloc) and `pfree` within the framework of memory contexts.

A `MemoryContext` is a region or scope where memory allocations occur. When a context is "reset" or "deleted," all memory allocated within that context (and, for deletion, its children) is automatically freed.

## Core Concepts

1.  **Hierarchy:** Memory contexts are organized in a tree. `TopMemoryContext` is the root. Each context (except `TopMemoryContext`) has a parent and can have multiple children.
2.  **`CurrentMemoryContext`:** A global variable that points to the memory context where `palloc()` allocations will occur by default.
3.  **Lifetimes:** Memory is typically allocated in contexts whose lifetime matches the data being stored. For example, per-query data is allocated in a query-lifespan context, and per-transaction data in a transaction-lifespan context.
4.  **Automatic Cleanup:** When a query or transaction ends, its associated memory context is typically reset or deleted, automatically freeing all memory allocated within it. This is a primary defense against memory leaks.
5.  **Context Types:** `MemoryContext` is an abstract type. PostgreSQL provides several concrete implementations, each optimized for different allocation patterns:
    *   `AllocSetContext`: General-purpose, most common. Allocates variable-sized chunks.
    *   `SlabContext`: For many allocations of the same, fixed size.
    *   `GenerationContext`: For objects with different lifespans within the same context.
    *   `BumpContext`: Simple, fast, sequential allocation; no individual free, only reset.

## `MemoryContextData` Structure

The `MemoryContextData` struct (aliased as `MemoryContext` for pointers) is the core representation:

```c
// From nodes/memnodes.h
typedef struct MemoryContextData
{
    NodeTag     type;           // Identifies exact kind of context (e.g., T_AllocSetContext)
    bool        isReset;        // True if no space allocated since last reset
    bool        allowInCritSection; // Allow palloc in critical section
    Size        mem_allocated;  // Track memory allocated for this context
    const MemoryContextMethods *methods; // Virtual function table for this context type
    MemoryContext parent;       // NULL if no parent (toplevel context)
    MemoryContext firstchild;   // Head of linked list of children
    MemoryContext prevchild;    // Previous child of same parent
    MemoryContext nextchild;    // Next child of same parent
    const char *name;           // Context name (usually a constant string)
    const char *ident;          // Context ID (optional, possibly dynamic)
    MemoryContextCallback *reset_cbs; // List of reset/delete callbacks
} MemoryContextData;
```

Each context type (e.g., `AllocSetContext`) will have its own struct that begins with these fields and then adds its own specific data.

### `MemoryContextMethods`
This struct within `MemoryContextData` acts as a virtual function table:
```c
// From nodes/memnodes.h
typedef struct MemoryContextMethods
{
    void       *(*alloc) (MemoryContext context, Size size, int flags);
    void        (*free_p) (void *pointer);
    void       *(*realloc) (void *pointer, Size size, int flags);
    void        (*reset) (MemoryContext context);
    void        (*delete_context) (MemoryContext context);
    MemoryContext (*get_chunk_context) (void *pointer);
    Size        (*get_chunk_space) (void *pointer);
    bool        (*is_empty) (MemoryContext context);
    void        (*stats) (MemoryContext context, ...);
#ifdef MEMORY_CONTEXT_CHECKING
    void        (*check) (MemoryContext context);
#endif
} MemoryContextMethods;
```
Each context implementation provides its own set of these functions.

## Key API Functions and Macros

### Context Creation
*   `MemoryContext AllocSetContextCreate(MemoryContext parent, const char *name, Size minContextSize, Size initBlockSize, Size maxBlockSize)`: Creates an `AllocSetContext`. Similar functions exist for other types (e.g., `SlabContextCreate`).
    *   `parent`: The parent context for the new context.
    *   `name`: A descriptive name (primarily for debugging).
    *   Other parameters control initial and incremental allocation sizes.
    *   Commonly used with predefined size macros like `ALLOCSET_DEFAULT_SIZES` or `ALLOCSET_SMALL_SIZES`.

### Allocation
*   `void *palloc(Size size)`: Allocates `size` bytes in the `CurrentMemoryContext`. The memory is zeroed if `ALLOCSET_DEFAULT_INITSIZE` is used for the context and the allocation is small enough, or explicitly by `palloc0`.
*   `void *palloc0(Size size)`: Allocates `size` bytes in `CurrentMemoryContext` and guarantees the memory is zero-filled.
*   `void pfree(void *pointer)`: Frees memory previously allocated by `palloc` or `palloc0`. The memory is returned to the context it was originally allocated from.
*   `MemoryContextSwitchTo(MemoryContext context)`: Sets `CurrentMemoryContext` to `context`. Returns the old `CurrentMemoryContext` so it can be restored.
    ```c
    MemoryContext oldcontext = MemoryContextSwitchTo(my_context);
    ptr = palloc(100);
    MemoryContextSwitchTo(oldcontext);
    ```
*   `void *MemoryContextAlloc(MemoryContext context, Size size)`: Allocates `size` bytes in the specified `context`, regardless of `CurrentMemoryContext`.
*   `char *MemoryContextStrdup(MemoryContext context, const char *string)`: Duplicates a string into the specified `context`.
*   `repalloc(void *pointer, Size size)`: Resizes a previously `palloc`'d chunk of memory.

### Context Manipulation
*   `void MemoryContextReset(MemoryContext context)`: Frees all memory allocated *within* `context` itself (not its children) since its creation or last reset. Reset callbacks are invoked. The context remains valid for new allocations.
*   `void MemoryContextDelete(MemoryContext context)`: Frees all memory allocated within `context` AND all its children, then destroys the context(s) themselves. Delete callbacks are invoked.
*   `void MemoryContextResetChildren(MemoryContext context)`: Calls `MemoryContextReset` on all direct children of `context`.
*   `void MemoryContextDeleteChildren(MemoryContext context)`: Calls `MemoryContextDelete` on all direct children of `context`.

### Inspection and Callbacks
*   `MemoryContextGetParent(MemoryContext context)`: Returns the parent of the given context.
*   `bool MemoryContextIsEmpty(MemoryContext context)`: Checks if any memory has been allocated in the context since it was created or last reset.
*   `void MemoryContextStats(MemoryContext context)`: Prints statistics about memory usage in `context` (and its children, recursively) to `stderr`.
*   `MemoryContextRegisterCallback(MemoryContext context, MemoryContextCallbackFunction func, Datum arg)`: Registers a callback function to be invoked when the context is reset or deleted.

## Standard Memory Contexts
PostgreSQL initializes several top-level memory contexts:
*   `TopMemoryContext`: The root of the context tree. Rarely used directly for allocations.
*   `ErrorContext`: Used for allocations made during error handling (e.g., building an error message). It's reset after each error.
*   `PostmasterContext`: Long-lived allocations in the postmaster process.
*   `CacheMemoryContext`: For system caches that should survive longer than individual transactions or queries (e.g., relation cache, plan cache).
*   `TopTransactionContext`: Parent context for all transaction-lifespan allocations.
*   `CurTransactionContext`: The current transaction's main memory context. Reset at transaction end.
*   `PortalContext`: A pointer that is dynamically updated to point to the memory context of the currently active portal (cursor). Query results are often allocated here.
*   `MessageContext`: Used for constructing messages to be sent to the client or server log. Typically reset after each message processing cycle.

## Usage Pattern

```c
#include "postgres.h"
#include "utils/memutils.h"
#include "utils/palloc.h"

void my_function_with_temp_data()
{
    MemoryContext my_local_context;
    MemoryContext old_context;
    char *buffer1;
    int *array_data;

    // 1. Create a new memory context as a child of CurrentMemoryContext
    //    This context will hold allocations specific to this function's operation.
    my_local_context = AllocSetContextCreate(CurrentMemoryContext,
                                             "My Local Function Context",
                                             ALLOCSET_SMALL_SIZES);

    // 2. Switch to the new context for allocations
    old_context = MemoryContextSwitchTo(my_local_context);

    // 3. Perform allocations. These will go into 'my_local_context'.
    buffer1 = (char *) palloc(100); // 100 bytes
    strcpy(buffer1, "Hello from my context!");

    array_data = (int *) palloc(sizeof(int) * 50); // Array of 50 ints
    for (int i = 0; i < 50; i++) array_data[i] = i;

    elog(LOG, "Buffer1: %s", buffer1);

    // 4. Restore the previous memory context
    MemoryContextSwitchTo(old_context);

    // 5. When done with all data in 'my_local_context', delete the context.
    //    This frees 'buffer1', 'array_data', and any other allocations made
    //    within 'my_local_context'.
    MemoryContextDelete(my_local_context);

    // buffer1 and array_data are now dangling pointers and should not be used.
}
```

## Benefits

*   **Leak Prevention:** Simplifies memory management by tying memory lifetime to context lifetime.
*   **Debugging:** Context names and hierarchy help in tracking down memory usage and leaks.
*   **Efficiency:** Different context types are optimized for different allocation patterns.
*   **Scoped Resource Management:** Callbacks allow for cleanup of other resources (e.g., file handles) tied to a context's lifetime.

This system is fundamental to PostgreSQL's stability and resource management.
