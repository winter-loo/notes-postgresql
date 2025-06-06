# PostgreSQL Bitmap Sets (`Bitmapset`)

**Source File:** `src/include/nodes/bitmapset.h`
**Implementation File:** `src/backend/nodes/bitmapset.c`

## Overview

A `Bitmapset` in PostgreSQL is a data structure designed to efficiently represent and manipulate sets of non-negative integers. It is particularly well-suited for sets where the maximum integer value is not excessively large (e.g., up to a few hundred or thousand). A key convention is that an empty `Bitmapset` is always represented by a `NULL` pointer.

Bitmapsets are used in various parts of PostgreSQL, such as representing sets of attribute numbers (column indices), relation OIDs, or other small integer collections.

## Structure Definition

```c
// The actual word size depends on the architecture (typically uint64 or uint32)
#if SIZEOF_VOID_P >= 8
#define BITS_PER_BITMAPWORD 64
typedef uint64 bitmapword;
#else
#define BITS_PER_BITMAPWORD 32
typedef uint32 bitmapword;
#endif

typedef struct Bitmapset
{
    NodeTag     type;       // T_Bitmapset
    int         nwords;     /* number of words in array */
    bitmapword  words[FLEXIBLE_ARRAY_MEMBER]; /* really [nwords] */
} Bitmapset;
```

*   `type`: The node type, always `T_Bitmapset`.
*   `nwords`: The number of `bitmapword`s allocated in the `words` array.
The size of the `words` array is determined by the largest integer present in the set. For example, if the largest member is 63, `nwords` would be 1 (on a 64-bit word system). If the largest member is 64, `nwords` would be 2.

## Key Concepts

*   **NULL for Empty:** An empty set is always represented by a `NULL` `Bitmapset*`.
*   **Memory Efficiency:** For dense sets or sets with relatively small maximum values, bitmapsets are very space-efficient.
*   **Bitwise Operations:** Set operations (union, intersection, difference) are implemented using fast bitwise logic.

## Common Operations

Many `Bitmapset` functions come in two flavors:
1.  Those that create a new `Bitmapset` for the result (e.g., `bms_union`).
2.  Those that modify one of their input `Bitmapset`s in-place, potentially reallocating or freeing it (e.g., `bms_add_member`, `bms_add_members`). These are often more efficient if the original set is no longer needed in its prior state.

### Initialization and Creation
*   `NULL`: Represents an empty set.
*   `Bitmapset *bms_make_singleton(int x)`: Creates a new bitmap set containing only the integer `x`.
*   `Bitmapset *bms_copy(const Bitmapset *a)`: Creates a new copy of bitmap set `a`.

### Adding and Removing Members
*   `Bitmapset *bms_add_member(Bitmapset *a, int x)`: Adds integer `x` to set `a`. `a` may be modified in-place or reallocated. Returns the (possibly new) `Bitmapset*`.
*   `Bitmapset *bms_del_member(Bitmapset *a, int x)`: Removes integer `x` from set `a`. `a` may be modified. Returns the (possibly new) `Bitmapset*`.
*   `Bitmapset *bms_add_members(Bitmapset *a, const Bitmapset *b)`: Adds all members of set `b` to set `a`. `a` is modified.
*   `Bitmapset *bms_del_members(Bitmapset *a, const Bitmapset *b)`: Removes all members of set `b` from set `a`. `a` is modified.
*   `Bitmapset *bms_add_range(Bitmapset *a, int lower, int upper)`: Adds all integers from `lower` to `upper` (inclusive) to set `a`. `a` is modified.

### Set Operations
*   `Bitmapset *bms_union(const Bitmapset *a, const Bitmapset *b)`: Returns a new set that is the union of `a` and `b`.
*   `Bitmapset *bms_intersect(const Bitmapset *a, const Bitmapset *b)`: Returns a new set that is the intersection of `a` and `b`.
*   `Bitmapset *bms_difference(const Bitmapset *a, const Bitmapset *b)`: Returns a new set containing elements in `a` but not in `b`.
*   `Bitmapset *bms_join(Bitmapset *a, Bitmapset *b)`: Equivalent to `bms_add_members(a, b)` followed by `bms_free(b)`. Modifies `a` and frees `b`.

### Membership and Properties
*   `bool bms_is_member(int x, const Bitmapset *a)`: Checks if integer `x` is a member of set `a`.
*   `bool bms_is_empty(const Bitmapset *a)`: Macro `((a) == NULL)`. Checks if set `a` is empty.
*   `int bms_num_members(const Bitmapset *a)`: Returns the number of members in set `a`.
*   `BMS_Membership bms_membership(const Bitmapset *a)`: Returns `BMS_EMPTY_SET`, `BMS_SINGLETON`, or `BMS_MULTIPLE`.
*   `int bms_singleton_member(const Bitmapset *a)`: If `a` is a singleton set, returns its member; otherwise, behavior is undefined or may error. Use `bms_get_singleton_member` for safer access.
*   `bool bms_get_singleton_member(const Bitmapset *a, int *member)`: If `a` is a singleton, stores the member in `*member` and returns `true`. Otherwise returns `false`.

### Comparison and Subsets
*   `bool bms_equal(const Bitmapset *a, const Bitmapset *b)`: Checks if sets `a` and `b` are equal.
*   `bool bms_is_subset(const Bitmapset *a, const Bitmapset *b)`: Checks if set `a` is a subset of set `b`.
*   `bool bms_overlap(const Bitmapset *a, const Bitmapset *b)`: Checks if sets `a` and `b` have any common members.
*   `BMS_Comparison bms_subset_compare(const Bitmapset *a, const Bitmapset *b)`: Compares two sets and returns `BMS_EQUAL`, `BMS_SUBSET1` (a is subset of b), `BMS_SUBSET2` (b is subset of a), or `BMS_DIFFERENT`.

### Iteration
*   `int bms_next_member(const Bitmapset *a, int prevbit)`: Finds the smallest member in set `a` that is greater than `prevbit`. To start iteration, `prevbit` should be -1. Returns -1 if no more members.
    ```c
    int member = -1;
    while ((member = bms_next_member(my_set, member)) >= 0)
    {
        // process member
    }
    ```
*   `int bms_prev_member(const Bitmapset *a, int prevbit)`: Finds the largest member in set `a` that is smaller than `prevbit`. To start iteration from the largest member, `prevbit` should be a value larger than any possible member (e.g., `bms_num_members(a)` if members are 0-indexed and dense, or a sufficiently large constant). Returns -2 if no more members. (Note: -2 is used to distinguish from a potential member -1 if that were allowed, though bitmapsets are for non-negative integers).

### Hashing
*   `uint32 bms_hash_value(const Bitmapset *a)`: Computes a hash value for the bitmap set `a`.
*   `uint32 bitmap_hash(const void *key, Size keysize)`: Hash function suitable for `HTAB` where the key is a `Bitmapset*`.
*   `int bitmap_match(const void *key1, const void *key2, Size keysize)`: Comparison function for `HTAB`.

### Memory Management
*   `void bms_free(Bitmapset *a)`: Frees the memory occupied by bitmap set `a`. Does nothing if `a` is `NULL`.
    Bitmapsets are typically allocated in a `MemoryContext`, and `bms_free` uses `pfree`. If the entire context is reset, explicit freeing is not always necessary.

## Usage Example

```c
#include "postgres.h"
#include "nodes/bitmapset.h"
#include "utils/memutils.h" // For palloc/pfree if not in a context

void example_bitmapset_usage()
{
    Bitmapset *set1 = NULL;
    Bitmapset *set2 = NULL;
    Bitmapset *union_set, *intersect_set;

    // Add members to set1: {1, 5, 10}
    set1 = bms_add_member(set1, 5);
    set1 = bms_add_member(set1, 1);
    set1 = bms_add_member(set1, 10);

    // Create set2: {5, 10, 20}
    set2 = bms_make_singleton(10);
    set2 = bms_add_member(set2, 20);
    set2 = bms_add_member(set2, 5);

    elog(LOG, "Set1 has %d members", bms_num_members(set1));
    elog(LOG, "Set2 has %d members", bms_num_members(set2));

    if (bms_is_member(5, set1))
    {
        elog(LOG, "5 is a member of Set1");
    }

    union_set = bms_union(set1, set2); // Union: {1, 5, 10, 20}
    elog(LOG, "Union has %d members", bms_num_members(union_set));

    intersect_set = bms_intersect(set1, set2); // Intersection: {5, 10}
    elog(LOG, "Intersection has %d members", bms_num_members(intersect_set));

    elog(LOG, "Iterating through union_set:");
    int member = -1;
    while ((member = bms_next_member(union_set, member)) >= 0)
    {
        elog(LOG, "  Member: %d", member);
    }

    // Free allocated sets
    bms_free(set1);
    bms_free(set2);
    bms_free(union_set);
    bms_free(intersect_set);
}
```

## Notes

*   Bitmapsets are 0-indexed. Negative numbers cannot be stored.
*   The performance of bitmapsets is best when the integers are relatively small and/or dense. For very sparse sets with large integer values, other representations might be more appropriate.
