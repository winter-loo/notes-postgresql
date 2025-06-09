# PostgreSQL Dynamic Strings (`StringInfo`)

**Source File:** `src/include/lib/stringinfo.h`
**Implementation File:** `src/common/stringinfo.c` (Note: some backend-specific aspects might be in `src/backend/lib/stringinfo.c` if it exists, but core logic is often in `common`)

## Overview

`StringInfoData` (commonly used via the `StringInfo` typedef, which is `StringInfoData*`) is PostgreSQL's fundamental data structure for creating and manipulating dynamically-sized strings. It provides an extensible buffer that can grow as needed, supporting both null-terminated C strings and arbitrary binary data. All memory is managed using `palloc()` in the backend (or `malloc()` in frontend utilities).

## Structure Definition

```c
typedef struct StringInfoData
{
    char       *data;     // Pointer to the string buffer
    int         len;      // Current length of the string (excluding NUL terminator for writable strings)
    int         maxlen;   // Allocated size of the 'data' buffer
    int         cursor;   // User-managed cursor/offset within the string, initialized to 0
} StringInfoData;

typedef StringInfoData *StringInfo;
```

*   `data`: The character buffer holding the string content.
*   `len`: The current number of bytes stored in `data`.
*   `maxlen`: The total allocated capacity of `data`. For writable `StringInfo` objects, `maxlen` is always greater than `len` to ensure space for a null terminator (`\0`).
*   `cursor`: An integer field initialized to zero. It is not directly used by most `StringInfo` manipulation functions but is available for callers to manage a current position if they are parsing or scanning the `StringInfo`'s content.

## Key Concepts

*   **Dynamic Sizing:** The internal buffer (`data`) automatically reallocates to accommodate more data when append functions are called.
*   **Null Termination:** For non-read-only `StringInfo` objects, the `data` buffer is guaranteed to have a terminating `\0` at `data[len]`.
*   **Read-Only Mode:** A `StringInfo` can be initialized to wrap an existing, possibly non-palloc'd, character buffer in a read-only fashion. In this mode, `maxlen` is set to 0, and append operations are disallowed.

## Initialization Methods

1.  **`StringInfo stringptr = makeStringInfo();`**
    *   Allocates both the `StringInfoData` structure itself and an initial internal data buffer using `palloc()`.
    *   This is suitable for `StringInfo` objects that need to persist beyond the current function scope and are managed on the heap.

2.  **`StringInfoData string; initStringInfo(&string);`**
    *   Initializes a `StringInfoData` structure that is typically allocated on the stack (or embedded in another struct).
    *   The internal data buffer is allocated using `palloc()`.
    *   This is common for `StringInfo` objects with a lifetime limited to the current function.

3.  **`StringInfoData string; initReadOnlyStringInfo(&string, char *existing_buffer, int buffer_len);`**
    *   Initializes `string.data` to point to `existing_buffer` and `string.len` to `buffer_len`.
    *   `string.maxlen` is set to 0, indicating it's read-only. No appends are allowed.
    *   The `existing_buffer` is not managed by the `StringInfo` (it might not be palloc'd) and does not need to be null-terminated if its usage doesn't require it.

4.  **`StringInfoData string; initStringInfoFromString(&string, char *palloced_buffer, int buffer_len);`**
    *   Initializes `string.data` to point to `palloced_buffer` (which must be a NUL-terminated, palloc'd buffer of size `buffer_len + 1`) and `string.len` to `buffer_len`.
    *   `string.maxlen` is set to `buffer_len + 1`.
    *   This `StringInfo` can be appended to, and `palloced_buffer` may be `repalloc`'d.

## Common Operations

### Appending Data
*   `void appendStringInfo(StringInfo str, const char *fmt, ...)`:
    *   Formats data according to the `sprintf`-style `fmt` string and appends it to `str->data`.
    *   Automatically enlarges the buffer if needed.
*   `void appendStringInfoString(StringInfo str, const char *s)`:
    *   Appends the null-terminated string `s` to `str->data`. Faster than `appendStringInfo(str, "%s", s)`.
*   `void appendStringInfoChar(StringInfo str, char ch)`:
    *   Appends a single character `ch` to `str->data`. Much faster than `appendStringInfo(str, "%c", ch)`.
*   `void appendStringInfoData(StringInfo str, const char *data, int datalen)`:
    *   Appends `datalen` bytes from the `data` buffer to `str->data`. This is suitable for binary data that may contain null bytes.

### Managing the Buffer
*   `void resetStringInfo(StringInfo str)`:
    *   Resets `str->len` to 0, effectively clearing the string content.
    *   The allocated buffer in `str->data` is retained for reuse, which is efficient if the `StringInfo` will be refilled shortly.
    *   Does not work on read-only `StringInfo`s.
*   `void enlargeStringInfo(StringInfo str, int needed)`:
    *   Ensures that `str->data` has enough space to append at least `needed` more bytes (plus a null terminator).
    *   If the current `str->maxlen - str->len - 1 < needed`, the buffer is reallocated (doubling its size or increasing by `needed`, whichever is larger).
    *   This is often called internally by append functions but can be called explicitly to preallocate space if a known large amount of data will be appended.

### Memory Management
*   `void destroyStringInfo(StringInfo str)`:
    *   Frees `str->data` and then frees the `StringInfoData` structure `str` itself.
    *   **Only use this for `StringInfo` objects created with `makeStringInfo()`**.
*   For `StringInfoData` objects initialized with `initStringInfo()` or `initStringInfoFromString()`, you typically free the data buffer with `pfree(str.data)` when it's no longer needed (if the `StringInfoData` itself is on the stack or part of a structure that will be freed separately).
*   For `initReadOnlyStringInfo()`, the caller is responsible for managing the lifetime of the external buffer.

## Usage Example

```c
#include "postgres.h"
#include "lib/stringinfo.h"
#include "utils/palloc.h" // For pfree

void example_stringinfo_usage()
{
    StringInfoData str_buf; // For stack-based StringInfoData
    StringInfo dyn_str;     // For heap-based StringInfo

    // --- Using initStringInfo (StringInfoData on stack) ---
    initStringInfo(&str_buf);

    appendStringInfoString(&str_buf, "Hello, ");
    appendStringInfoChar(&str_buf, 'W');
    appendStringInfo(&str_buf, "orld! Count: %d", 123);

    elog(LOG, "str_buf: %s (len: %d, maxlen: %d)", str_buf.data, str_buf.len, str_buf.maxlen);

    // To reuse str_buf:
    resetStringInfo(&str_buf);
    appendStringInfoString(&str_buf, "New content.");
    elog(LOG, "str_buf after reset: %s", str_buf.data);

    pfree(str_buf.data); // Free the palloc'd buffer

    // --- Using makeStringInfo (StringInfoData on heap) ---
    dyn_str = makeStringInfo();
    appendStringInfoString(dyn_str, "Dynamic string example.");
    appendStringInfoData(dyn_str, "\0Binary\0Data", 12); // Append binary data

    elog(LOG, "dyn_str len: %d, data[0-15]: %.15s...", dyn_str->len, dyn_str->data);

    destroyStringInfo(dyn_str); // Frees dyn_str->data and dyn_str itself

    // --- Using initReadOnlyStringInfo ---
    StringInfoData read_only_str;
    const char *my_const_string = "This is read-only";
    initReadOnlyStringInfo(&read_only_str, (char *)my_const_string, strlen(my_const_string));
    // Cannot append to read_only_str
    // No need to free read_only_str.data if my_const_string is a literal or managed elsewhere
    elog(LOG, "Read-only: %.*s", read_only_str.len, read_only_str.data);
}
```

## Notes

*   `StringInfo` is a cornerstone for string construction in PostgreSQL, used in query parsing, message formatting, and many other areas.
*   Be mindful of the initialization method to correctly manage memory (especially when to use `destroyStringInfo` vs. just `pfree(str->data)`).
*   The `cursor` field is available for custom iteration or parsing logic but is not modified by the standard append or reset functions.
