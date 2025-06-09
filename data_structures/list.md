# PostgreSQL Generic List (`List`)

**Source File:** `src/include/nodes/pg_list.h`

## Overview

The `List` data structure in PostgreSQL is a generic, expansible array used extensively throughout the codebase. It evolved from an earlier Lisp-style cons-cell list implementation but is now optimized for C. An empty list is always represented by a `NIL` pointer (defined as `((List *) NULL)`).

## Types of Lists

PostgreSQL's `List` can store different types of elements, distinguished by a `NodeTag` in the list header:

*   `T_List`: Stores `void *` pointers. This is the most common type, often used for lists of `Node` structures.
*   `T_IntList`: Stores `int` values.
*   `T_OidList`: Stores `Oid` (Object Identifier) values.
*   `T_XidList`: Stores `TransactionId` values.

## Structure Definition

```c
typedef union ListCell
{
    void       *ptr_value;
    int         int_value;
    Oid         oid_value;
    TransactionId xid_value;
} ListCell;

typedef struct List
{
    NodeTag     type;           /* T_List, T_IntList, T_OidList, or T_XidList */
    int         length;         /* number of elements currently present */
    int         max_length;     /* allocated length of elements[] */
    ListCell   *elements;       /* re-allocatable array of cells */
    /* We may allocate some cells along with the List header: */
    ListCell    initial_elements[FLEXIBLE_ARRAY_MEMBER];
    /* If elements == initial_elements, it's not a separate allocation */
} List;
```

*   `type`: Specifies the type of elements in the list.
*   `length`: The current number of elements in the list.
*   `max_length`: The current allocated capacity of the `elements` array.
*   `elements`: A pointer to the dynamically allocated array of `ListCell`s.
*   `initial_elements`: For small lists, elements can be stored directly within the `List` struct itself to avoid a separate allocation. If `elements == initial_elements`, the data is stored inline.

## Key Concepts

*   **NIL:** An empty list is *always* represented by `NIL` (a null pointer). A non-NIL list is guaranteed to have `length >= 1`.
*   **Expansible Array:** Internally, lists are implemented as arrays that can grow as needed.
*   **ListCell:** A union that holds the actual data for each element, depending on the list type.

## Common Operations and Macros

PostgreSQL provides a rich set of functions and macros for manipulating lists. Here are some of the most common ones:

### Initialization and Creation
*   `NIL`: Represents an empty list.
*   `list_make1(datum)`: Creates a new `T_List` with one `void*` element.
*   `list_make1_int(datum)`: Creates a new `T_IntList` with one integer element.
*   `list_make1_oid(datum)`: Creates a new `T_OidList` with one OID element.
*   Similar `list_make2`, `list_make3`, etc., exist for creating lists with multiple initial elements.

### Adding Elements
*   `lappend(list, datum)`: Appends a `void*` element to a `T_List`. Returns the (possibly new) list pointer.
*   `lappend_int(list, datum)`: Appends an integer to a `T_IntList`.
*   `lappend_oid(list, datum)`: Appends an OID to a `T_OidList`.
*   `lcons(datum, list)`: Prepends a `void*` element to a `T_List`. Returns the new list pointer.
*   `lcons_int(datum, list)`: Prepends an integer to a `T_IntList`.
*   `lcons_oid(datum, list)`: Prepends an OID to a `T_OidList`.

### Accessing Elements
*   `list_length(list)`: Returns the number of elements in the list (0 if `NIL`).
*   `list_head(list)`: Returns a pointer to the first `ListCell` (or `NULL` if empty).
*   `list_tail(list)`: Returns a pointer to the last `ListCell` (or `NULL` if empty).
*   `linitial(list)`: Returns the `void*` data from the first cell.
*   `linitial_int(list)`: Returns the integer data from the first cell.
*   `linitial_oid(list)`: Returns the OID data from the first cell.
*   `lsecond(list)`, `lthird(list)`, `lfourth(list)`: Access data from the 2nd, 3rd, 4th cells.
*   `llast(list)`: Returns the `void*` data from the last cell.
*   `list_nth(list, n)`: Returns the `void*` data from the n-th cell (0-indexed).
*   `list_nth_int(list, n)`, `list_nth_oid(list, n)`: Similar for int and OID lists.
*   `list_nth_cell(list, n)`: Returns a pointer to the n-th `ListCell`.

### Iteration
*   `foreach(cell, list)`: A macro to iterate through a `T_List`. `cell` is a `ListCell*` loop variable.
    ```c
    ListCell *lc;
    foreach(lc, my_list)
    {
        void *datum = lfirst(lc); // Get ptr_value from cell
        // ... process datum ...
    }
    ```
*   `forboth(cell1, list1, cell2, list2)`: Iterates through two lists simultaneously, as long as both have elements.
*   Many other specialized iteration macros exist (e.g., `for_each_from`, `for_each_cell_setup`).

### Modifying Lists
*   `list_concat(list1, list2)`: Concatenates `list2` onto `list1`. `list2` is not modified.
*   `list_truncate(list, new_len)`: Truncates a list to `new_len` elements.
*   `list_delete(list, cell)`: Deletes the specified cell from the list. (More complex: `list_delete_ptr`, `list_delete_first`, `list_delete_last`).

### Searching and Membership
*   `list_member(list, datum)`: Checks if a `void*` datum is in a `T_List` (pointer comparison).
*   `list_member_ptr(list, datum)`: Same as `list_member`.
*   `list_member_int(list, datum)`: Checks if an integer is in a `T_IntList`.
*   `list_member_oid(list, datum)`: Checks if an OID is in a `T_OidList`.

### Copying
*   `list_copy(list)`: Creates a shallow copy of the list header and elements array.
*   `list_copy_tail(list, nskip)`: Copies the tail of a list, skipping the first `nskip` elements.

### Unique Element Lists
*   `list_append_unique_ptr(list, datum)`: Appends `datum` only if it's not already present (pointer comparison).
*   `list_append_unique_int(list, datum)`, `list_append_unique_oid(list, datum)`: Similar for int and OID lists.
*   `list_concat_unique(list1, list2)`: Concatenates, adding only unique elements from `list2`.

## Usage Considerations

*   **Memory Management:** Lists are allocated using `palloc`. When a list is no longer needed, the memory context it was allocated in should be freed or reset. Individual elements (if they are pointers to palloc'd structures) might need separate freeing if not part of the same memory context lifecycle.
*   **Pointer Stability:** The `List` header itself is stable as long as the list is not empty. However, the `elements` array can be reallocated if the list grows, so pointers to `ListCell`s should not be held across operations that might modify the list's length or capacity.
*   **Type Safety:** While `T_List` stores `void*`, it's crucial for the programmer to know the actual type of data being stored and cast appropriately. Macros like `lfirst_node(type, lc)` can help with casting `Node` pointers.

## Example

```c
#include "nodes/pg_list.h"
#include "nodes/parsenodes.h" // For an example Node type like TargetEntry
#include "utils/palloc.h"

// ... within a function ...

List *my_target_list = NIL;
TargetEntry *tle1, *tle2;

// Assume tle1 and tle2 are palloc'd and initialized TargetEntry structs
tle1 = makeNode(TargetEntry);
// ... initialize tle1 ...

tle2 = makeNode(TargetEntry);
// ... initialize tle2 ...

my_target_list = lappend(my_target_list, tle1);
my_target_list = lappend(my_target_list, tle2);

if (list_length(my_target_list) == 2)
{
    // This will be true
}

ListCell *lc;
foreach(lc, my_target_list)
{
    TargetEntry *current_tle = (TargetEntry *) lfirst(lc);
    // Process current_tle
}

// If my_target_list was allocated in CurrentMemoryContext, it will be freed
// when CurrentMemoryContext is reset or deleted.
```
