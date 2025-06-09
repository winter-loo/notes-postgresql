# PostgreSQL Dynamic Hash Tables (`HTAB`)

**Source Header:** `src/include/utils/hsearch.h`
**Implementation File:** `src/backend/utils/hash/dynahash.c`

## Overview

PostgreSQL's dynamic hash tables, commonly referred to as `HTAB`, provide an efficient mechanism for key-based data lookup, insertion, and deletion. They are designed to be flexible, allowing for custom key types, hash functions, comparison routines, and memory management. These hash tables can dynamically resize themselves to maintain good performance as the number of elements changes.

`HTAB` is used extensively throughout the PostgreSQL backend for various purposes, such as caching, managing session-specific data, and implementing join algorithms.

## Key Structures and Types

### `HTAB`
An opaque structure representing the hash table itself. Pointers to `HTAB` are returned by `hash_create()` and used in subsequent operations.

### `HASHCTL`
```c
typedef struct HASHCTL
{
    /* Used if HASH_PARTITION flag is set: */
    long        num_partitions; /* # partitions (must be power of 2) */
    /* Used if HASH_SEGMENT flag is set: */
    long        ssize;          /* segment size */
    /* Used if HASH_DIRSIZE flag is set: */
    long        dsize;          /* (initial) directory size */
    long        max_dsize;      /* limit to dsize if dir size is limited */
    /* Used if HASH_ELEM flag is set (which is now required): */
    Size        keysize;        /* hash key length in bytes */
    Size        entrysize;      /* total user element size in bytes */
    /* Used if HASH_FUNCTION flag is set: */
    HashValueFunc hash;         /* hash function */
    /* Used if HASH_COMPARE flag is set: */
    HashCompareFunc match;      /* key comparison function */
    /* Used if HASH_KEYCOPY flag is set: */
    HashCopyFunc keycopy;       /* key copying function */
    /* Used if HASH_ALLOC flag is set: */
    HashAllocFunc alloc;        /* memory allocator */
    /* Used if HASH_CONTEXT flag is set: */
    MemoryContext hcxt;         /* memory context to use for allocations */
    /* Used if HASH_SHARED_MEM flag is set: */
    HASHHDR    *hctl;           /* location of header in shared mem */
} HASHCTL;
```
This structure is used to pass parameters to `hash_create()`. The `flags` argument to `hash_create()` determines which fields of `HASHCTL` are considered.

*   `keysize`: The size of the hash key in bytes.
*   `entrysize`: The total size of the user's data element in bytes. The key is typically expected to be at the beginning of this entry.
*   `hash`: Pointer to the hash function.
*   `match`: Pointer to the key comparison function.
*   `keycopy`: Pointer to the key copying function (used if keys are not stored in-place or need special handling).
*   `alloc`: Pointer to a custom memory allocation function (e.g., `malloc`).
*   `hcxt`: A `MemoryContext` to use for all allocations if `alloc` is not provided. This is the common way to manage memory for hash tables.
*   Other fields control initial sizing, partitioning (for shared memory), and shared memory setup.

### `HASHELEMENT`
```c
typedef struct HASHELEMENT
{
    struct HASHELEMENT *link;   /* link to next entry in same bucket */
    uint32      hashvalue;      /* hash function result for this entry */
} HASHELEMENT;
```
This is an internal header structure for each entry stored in the hash table. The user's actual data (of `entrysize` bytes) follows this header in memory. The `key` is assumed to be at the start of the user's data.

### `HASHACTION`
```c
typedef enum
{
    HASH_FIND,      // Look for an existing element
    HASH_ENTER,     // Look for an existing element, or enter a new one if not found
    HASH_REMOVE,    // Look for an existing element and remove it
    HASH_ENTER_NULL // Like HASH_ENTER, but returns NULL if element already exists and sets found=true
} HASHACTION;
```
This enum specifies the operation to be performed by `hash_search()`.

### Function Pointer Types
*   `HashValueFunc`: `uint32 (*HashValueFunc) (const void *key, Size keysize);`
    *   Computes a hash value for the given key.
*   `HashCompareFunc`: `int (*HashCompareFunc) (const void *key1, const void *key2, Size keysize);`
    *   Compares two keys. Returns 0 for a match, non-zero otherwise (compatible with `memcmp`, `strncmp`).
*   `HashCopyFunc`: `void *(*HashCopyFunc) (void *dest, const void *src, Size keysize);`
    *   Copies a key. (Compatible with `memcpy`).
*   `HashAllocFunc`: `void *(*HashAllocFunc) (Size request);`
    *   Allocates memory (compatible with `malloc`).

## Core API Functions

### Creation and Destruction
*   `HTAB *hash_create(const char *tabname, long nelem, const HASHCTL *info, int flags)`
    *   Creates a new hash table.
    *   `tabname`: A name for the hash table (used in error messages).
    *   `nelem`: An estimate of the maximum number of elements, used for initial sizing.
    *   `info`: Pointer to a `HASHCTL` structure containing configuration parameters.
    *   `flags`: Bitmask indicating which `HASHCTL` fields are valid and other options.
        *   `HASH_ELEM`: **Required.** Specifies that `keysize` and `entrysize` are provided.
        *   `HASH_FUNCTION`: Use custom hash function from `info->hash`.
        *   `HASH_COMPARE`: Use custom comparison function from `info->match`.
        *   `HASH_KEYCOPY`: Use custom key copy function from `info->keycopy`.
        *   `HASH_ALLOC`: Use custom allocator from `info->alloc`.
        *   `HASH_CONTEXT`: Allocate memory in `info->hcxt`. If not set and `HASH_ALLOC` is not set, `TopMemoryContext` is used.
        *   `HASH_STRINGS`: Use built-in support for null-terminated string keys (provides default hash, compare, and copy functions).
        *   `HASH_BLOBS`: Use built-in support for binary blob keys (provides default hash, compare using `memcmp`, and copy using `memcpy`).
        *   `HASH_SHARED_MEM`: Table is in shared memory. Requires `info->hctl` to be set.
        *   `HASH_FIXED_SIZE`: The initial size is a hard limit; the table will not expand.
*   `void hash_destroy(HTAB *hashp)`
    *   Frees all memory associated with the hash table, including all its elements. If a `MemoryContext` was used, often destroying the context is sufficient.

### Searching, Inserting, and Deleting
*   `void *hash_search(HTAB *hashp, const void *keyPtr, HASHACTION action, bool *foundPtr)`
    *   The primary function for interacting with the hash table.
    *   `hashp`: Pointer to the hash table.
    *   `keyPtr`: Pointer to the key to search for/insert/delete.
    *   `action`: The operation to perform (`HASH_FIND`, `HASH_ENTER`, `HASH_REMOVE`, `HASH_ENTER_NULL`).
    *   `foundPtr`: (Output parameter) Set to `true` if an element with the given key was found, `false` otherwise.
    *   **Return Value**:
        *   If `action` is `HASH_FIND` or `HASH_REMOVE`: Returns a pointer to the found element, or `NULL` if not found.
        *   If `action` is `HASH_ENTER`: Returns a pointer to the (possibly new) element. If a new element is created, `*foundPtr` will be `false`. If an existing element is found, `*foundPtr` will be `true`.
        *   If `action` is `HASH_ENTER_NULL`: If the element exists, `*foundPtr` is true and `NULL` is returned. If the element does not exist, a new entry is created, `*foundPtr` is false, and a pointer to the new entry is returned.
    *   When a new element is entered (`HASH_ENTER` and `*foundPtr` is false, or `HASH_ENTER_NULL` and `*foundPtr` is false), the space for the element is allocated and zeroed. The key is copied from `keyPtr` into the new element if `HASH_KEYCOPY` was specified or if default key copying is active (e.g. `HASH_STRINGS`). If you are managing the key within the `entrysize` yourself and not using `HASH_KEYCOPY`, you might need to populate the key and other fields after `hash_search` returns.

*   `void *hash_search_with_hash_value(HTAB *hashp, const void *keyPtr, uint32 hashvalue, HASHACTION action, bool *foundPtr)`
    *   Similar to `hash_search`, but allows providing a precomputed hash value. This can be useful if the hash value is expensive to compute and is needed multiple times.
*   `uint32 get_hash_value(HTAB *hashp, const void *keyPtr)`
    *   Computes and returns the hash value for `keyPtr` using the table's configured hash function.

### Iteration
*   `void hash_seq_init(HASH_SEQ_STATUS *status, HTAB *hashp)`
    *   Initializes a scan of the hash table.
    *   `status`: Pointer to a `HASH_SEQ_STATUS` struct (caller-allocated) to hold scan state.
*   `void *hash_seq_search(HASH_SEQ_STATUS *status)`
    *   Returns the next element in the hash table scan, or `NULL` if no more elements.
*   `void hash_seq_term(HASH_SEQ_STATUS *status)`
    *   Cleans up after a scan. Not strictly necessary if the `HASH_SEQ_STATUS` is on the stack and will go out of scope, but good practice.

### Other Utility Functions
*   `long hash_get_num_entries(HTAB *hashp)`: Returns the current number of elements in the table.
*   `void hash_stats(const char *where, HTAB *hashp)`: Prints statistics about the hash table to `stderr` (useful for debugging performance).
*   `void hash_freeze(HTAB *hashp)`: Optimizes the table for read-only access after population (can reduce memory footprint).
*   `Size hash_estimate_size(long num_entries, Size entrysize)`: Estimates memory needed for a hash table.

## Usage Considerations

*   **Memory Management:** Typically, hash tables are allocated within a specific `MemoryContext`. When the context is reset or deleted, the hash table and all its elements are automatically freed. If a custom allocator (`HASH_ALLOC`) is used, the user is responsible for managing memory.
*   **Key Structure:** The key is generally expected to be at the beginning of the `entrysize` data block for each element. If `HASH_STRINGS` or `HASH_BLOBS` is used, or a custom `keycopy` function, the system handles copying the key into the allocated entry. Otherwise, the user might need to copy the key into the returned element structure after a successful `HASH_ENTER` for a new element.
*   **Choosing Hash and Compare Functions:** The quality of the hash function is critical for performance. PostgreSQL provides several stock hash functions (e.g., `string_hash`, `tag_hash`, `uint32_hash`). The comparison function must correctly determine equality.
*   **Concurrency:** For hash tables in shared memory, `HASH_PARTITION` can be used to enable partitioned locking, reducing contention.
*   **Error Handling:** Always check the `foundPtr` output parameter after `hash_search` to understand the outcome of the operation.

## Example

```c
#include "postgres.h"
#include "utils/hsearch.h"
#include "utils/memutils.h" // For MemoryContexts

// Define a structure for our hash table entries
typedef struct MyHashEntry
{
    char    key[32];    // Key (null-terminated string)
    int     value;      // Some data associated with the key
    // HASHELEMENT must be effectively first if not using HASH_BLOBS/HASH_STRINGS
    // but with HASH_STRINGS, the key is handled by the HTAB itself.
} MyHashEntry;

void example_hash_table_usage()
{
    HTAB *my_hash_table;
    HASHCTL ctl;
    MyHashEntry *entry;
    bool found;

    // Initialize HASHCTL structure
    MemSet(&ctl, 0, sizeof(ctl));
    ctl.keysize = sizeof(char[32]); // Or just strlen + 1 for actual strings if not fixed size
    ctl.entrysize = sizeof(MyHashEntry);
    ctl.hcxt = CurrentMemoryContext; // Use current memory context
    // HASH_STRINGS will use string_hash, strncmp-like comparison, and strcpy-like key copy.
    // It assumes the key is the first field of entrysize.

    my_hash_table = hash_create("My Test Hash Table",
                                100,  // Initial estimated size
                                &ctl,
                                HASH_ELEM | HASH_STRINGS | HASH_CONTEXT);

    // --- Insert an element ---
    const char *key1 = "example_key_1";
    entry = (MyHashEntry *) hash_search(my_hash_table,
                                        key1,
                                        HASH_ENTER,
                                        &found);
    if (!found) {
        // New entry was created, key was copied by HASH_STRINGS mechanism
        // If not using HASH_STRINGS or HASH_KEYCOPY, you'd copy the key here:
        // strncpy(entry->key, key1, sizeof(entry->key) -1 ); entry->key[sizeof(entry->key)-1] = '\0';
        entry->value = 123;
        elog(LOG, "Inserted key: %s, value: %d", entry->key, entry->value);
    } else {
        elog(LOG, "Key %s already existed.", key1);
    }

    // --- Insert another element ---
    const char *key2 = "another_key_22";
    entry = (MyHashEntry *) hash_search(my_hash_table,
                                        key2,
                                        HASH_ENTER,
                                        &found);
    if (!found) {
        entry->value = 456;
        elog(LOG, "Inserted key: %s, value: %d", entry->key, entry->value);
    }

    // --- Find an element ---
    const char *search_key = "example_key_1";
    entry = (MyHashEntry *) hash_search(my_hash_table,
                                        search_key,
                                        HASH_FIND,
                                        &found);
    if (found) {
        elog(LOG, "Found key: %s, value: %d", entry->key, entry->value);
    } else {
        elog(LOG, "Key %s not found.", search_key);
    }

    // --- Iterate over the hash table ---
    HASH_SEQ_STATUS hash_seq;
    elog(LOG, "Iterating over hash table:");
    hash_seq_init(&hash_seq, my_hash_table);
    while ((entry = (MyHashEntry *) hash_seq_search(&hash_seq)) != NULL)
    {
        elog(LOG, "  Key: %s, Value: %d", entry->key, entry->value);
    }
    // hash_seq_term(&hash_seq); // Optional if status is on stack

    // --- Remove an element ---
    const char *remove_key = "another_key_22";
    entry = (MyHashEntry *) hash_search(my_hash_table,
                                        remove_key,
                                        HASH_REMOVE,
                                        &found);
    if (found) {
        elog(LOG, "Removed key: %s", remove_key); // entry pointer is valid here
    } else {
        elog(LOG, "Key %s not found for removal.", remove_key);
    }

    // No need to call hash_destroy if CurrentMemoryContext will be reset/deleted.
    // Otherwise: hash_destroy(my_hash_table);
}

```
