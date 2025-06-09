# PostgreSQL pg_dump Utility Modification Guide

## Overview

This guide provides essential information for developers who want to modify the `pg_dump` utility, PostgreSQL's database backup tool. Understanding the architecture, key APIs, and concepts will help you make effective changes to the codebase.

## Architecture

`pg_dump` is a command-line utility that extracts a PostgreSQL database into a script file or other archive format. Its architecture consists of several components:

- **Schema Extraction**: Reads system catalogs to determine database structure
- **Data Extraction**: Extracts table data in various formats
- **Archive Creation**: Creates output in various formats (plain text, custom, directory, or tar)
- **Compression**: Supports various compression methods
- **Parallelism**: Can perform operations in parallel for better performance

## Key Files and Their Purposes

The source code for `pg_dump` is located in `src/bin/pg_dump/`:

| File | Description |
|------|-------------|
| `pg_dump.c` | Core functionality and main entry point (~600KB) |
| `pg_dump.h` | Type definitions and function declarations |
| `pg_backup.h` | Archive format definitions and interfaces |
| `pg_backup_archiver.c` | Archive file handling and management |
| `pg_backup_archiver.h` | Archive function declarations |
| `pg_backup_db.c` | Database connection and query handling |
| `pg_backup_tar.c` | Tar format specific functions |
| `pg_backup_custom.c` | Custom format specific functions |
| `pg_backup_directory.c` | Directory format specific functions |
| `pg_backup_null.c` | NULL (test) output format |
| `common.c` | Common utility functions |
| `dumputils.c` | Utilities for dumping objects |
| `parallel.c` | Parallel processing infrastructure |
| `compress_io.c` | Compression/decompression handling |
| `filter.c` | Object filtering functionality |

## Core Concepts and APIs

### Archive Structure

At the core of pg_dump is the `Archive` struct, which defines the interface for all operations:

```c
typedef struct Archive
{
    /* AH is short for "archive handle" */
    ArchiveFormat format;    /* archive format */
    ArchiveMode   mode;      /* archive mode (read or write) */
    
    /* Various callbacks for archive operations */
    WriteFunc     WritePtr;  /* Write data to archive */
    ReadFunc      ReadPtr;   /* Read data from archive */
    /* ... many other function pointers ... */
    
    /* Archive file format specific data */
    void        *formatData; /* private data for format handlers */
    
    /* Database connection */
    PGconn      *connection;
    
    /* Various state information */
    /* ... */
} Archive;
```

### DumpableObject System

Objects are represented by the `DumpableObject` structure and specialized subtypes:

```c
typedef struct _dumpableObject
{
    DumpId        dumpId;         /* Unique ID for cross-references */
    ObjectType    objType;        /* Type of object */
    char       *name;           /* Object name */
    char       *namespace;      /* Namespace, or NULL if none */
    char       *owner;          /* Owner, or NULL if not applicable */
    bool        dump;           /* True to dump this object */
    /* ... other fields ... */
} DumpableObject;
```

Common subtypes include:
- `TableInfo` - Tables and similar objects
- `TypeInfo` - Data types
- `FuncInfo` - Functions and procedures
- `IndxInfo` - Indexes
- `NamespaceInfo` - Schemas/namespaces

### libpq Interface

Like `psql`, `pg_dump` uses libpq for database connections:

```c
PGconn *conn = PQconnectdb(connstr);
PGresult *res = PQexec(conn, query);
// Process results...
PQclear(res);
// Later...
PQfinish(conn);
```

### Options and Settings

Settings are stored in a `DumpOptions` structure and passed around:

```c
typedef struct _dumpOptions
{
    bool        schemaOnly;    /* dump schema only */
    bool        dataOnly;      /* dump data only */
    bool        blobs;         /* include large objects */
    /* ... many more options ... */
} DumpOptions;
```

### Output Formats

pg_dump supports multiple output formats:
- **Plain text** SQL script
- **Custom** PostgreSQL's custom binary format
- **Directory** Split into multiple files in a directory
- **Tar** Tarball containing split files

### Error Handling

`pg_dump` uses the following for error handling:
- `pg_fatal()` - Log message and exit program
- `pg_log_error()` - Log error message but continue
- `pg_log_warning()` - Log warning message
- `pg_log_info()` - Log informational message

## Common Modification Patterns

### Adding a New Object Type

1. Define a new struct to hold object-specific information
2. Add a new ObjectType in `pg_dump.h`
3. Create functions to:
   - Extract object information from database (`get<Object>s()`)
   - Dump object definition (`dump<Object>()`)
   - Handle object dependencies
4. Register handler in the appropriate initialization function

### Adding Command Line Options

1. Modify `main()` in `pg_dump.c` to add the option to the `getopt_long` array
2. Add appropriate case in the switch statement that processes options
3. Store the setting in the appropriate variable or `DumpOptions` field
4. Use the setting during dump operations

### Modifying Output Format

1. Identify the function that generates the output you want to modify
2. Update the function to include additional information or formatting
3. Ensure changes work across all supported archive formats

### Adding Dependency Tracking

1. Identify the object relationship that creates the dependency
2. Add appropriate calls to `addObjectDependency()` during object creation
3. Test that objects are dumped in the proper order

## Key Workflows

### Schema Extraction Process

1. Connect to database (`setup_connection()`)
2. Read system catalogs to identify objects
   - Each object type has a `get<Objects>()` function (e.g., `getTables()`)
3. Build dependency graph between objects
4. Sort objects for dump based on dependencies
5. Output objects in sorted order

### Data Extraction Process

1. For each table to be dumped:
   - Generate SQL query to get table data
   - Format the data based on output type
   - Write data to archive using appropriate format handler

### Filtering System

The filtering system determines which objects should be included:
1. Command-line options like `--schema` and `--table` populate pattern lists
2. Pattern lists are expanded to OID lists (`expand_<object>_name_patterns()`)
3. Selection functions (`selectDumpable<Object>()`) determine if each object should be dumped
4. Objects marked to skip are excluded from the output

## Building and Testing

To build pg_dump after making changes:

```bash
cd src/bin/pg_dump
make
make install
```

For testing:
- Create test dumps of your databases to verify output
- Check restored database integrity
- Run regression tests in `src/bin/pg_dump/t/`

## Debugging Tips

- Use `--verbose` to see more information about the dump process
- Examine the system catalog queries with `PGDEBUG=1` environment variable
- Use a custom format dump and check its contents with `pg_restore -l`
- Test with both large and small databases to catch performance issues

## Common Pitfalls

- Always check for NULL pointers before dereferencing
- Be careful with memory management (use appropriate allocation/free functions)
- Handle quoting and escaping correctly for SQL output
- Consider backward compatibility with older PostgreSQL versions
- Test with databases containing special characters and edge cases
- Be careful with dependencies to ensure objects are created in correct order

## Additional Resources

- PostgreSQL source code documentation: https://doxygen.postgresql.org/
- PostgreSQL developer mailing lists: https://www.postgresql.org/list/
- libpq documentation: https://www.postgresql.org/docs/current/libpq.html
- PostgreSQL system catalog documentation: https://www.postgresql.org/docs/current/catalogs.html
- Existing test cases in `src/bin/pg_dump/t/`
