# PostgreSQL psql Client Modification Guide

## Overview

This guide provides essential information for developers who want to modify the `psql` binary program, PostgreSQL's interactive terminal client. Understanding the architecture, key APIs, and concepts will help you make effective modifications to the codebase.

## Architecture

`psql` is a command-line client application that communicates with PostgreSQL servers. Its architecture consists of several components:

- **Command Parser**: Processes commands prefixed with backslashes (`\`)
- **SQL Parser**: Handles SQL statements
- **Input/Output System**: Manages user input and result display
- **Connection Management**: Handles connections to PostgreSQL servers
- **Readline Integration**: Provides command history and tab completion

## Key Files and Their Purposes

The source code for `psql` is located in `src/bin/psql/`:

| File | Description |
|------|-------------|
| `startup.c` | Contains the `main()` function and initialization logic |
| `mainloop.c` | Implements the main processing loop for reading and executing commands |
| `command.c/h` | Handles backslash (`\`) commands |
| `common.c/h` | Common utility functions |
| `input.c/h` | Input handling |
| `output.c` | Output formatting |
| `describe.c/h` | Implements `\d` commands for describing database objects |
| `tab-complete.c` | Implements tab completion |
| `variables.c/h` | Manages psql variables |
| `prompt.c/h` | Handles command prompt logic |
| `settings.h` | Global settings and options |

## Core Concepts and APIs

### PQExpBuffer

The `PQExpBuffer` is used throughout psql for string manipulation. Key functions:
- `createPQExpBuffer()` - Create a new buffer
- `resetPQExpBuffer()` - Clear the buffer contents
- `appendPQExpBuffer()` - Append formatted text to buffer
- `terminatePQExpBuffer()` - Ensure null termination
- `destroyPQExpBuffer()` - Free buffer memory

Example:
```c
PQExpBuffer buf = createPQExpBuffer();
appendPQExpBuffer(buf, "SELECT * FROM %s", tablename);
// use buf->data to access the string
destroyPQExpBuffer(buf);
```

### libpq Connection

`psql` uses libpq for database connections, with the connection pointer stored in `pset.db`:

- `PQconnectdb()` - Connect to database
- `PQexec()` - Execute a command
- `PQresultStatus()` - Get command result status
- `PQclear()` - Free result memory
- `PQfinish()` - Close connection

Example:
```c
PGresult *res = PQexec(pset.db, query);
if (PQresultStatus(res) != PGRES_TUPLES_OK)
{
    // handle error
}
PQclear(res);
```

### Global Settings: PsqlSettings

The global `pset` variable (of type `PsqlSettings`) contains all configuration and state:

```c
extern PsqlSettings pset;
```

Key fields:
- `pset.db` - Active connection
- `pset.queryFout` - Output stream
- `pset.popt` - Print options
- `pset.vars` - psql variables

### Command Processing

Commands are processed in `mainloop.c` which handles:
- Reading input
- Processing backslash commands
- Sending SQL to server
- Handling results

Backslash commands are implemented in `command.c` with `HandleSlashCmds()` dispatching to appropriate handlers.

### Input/Output

- Input functions in `input.c` support both interactive and file-based input
- Output formatting in `print.c` provides table, aligned, and various output formats
- `PrintQueryResults()` formats and displays query results

## Common Modification Patterns

### Adding a New Backslash Command

1. Add new command to `HandleSlashCmds()` in `command.c`
2. Implement the command logic in a new function
3. Update help text in `help.c`
4. Add tab completion support in `tab-complete.in.c` if needed

### Modifying Output Format

1. Adjust the print options in the `printQueryOpt` struct
2. Modify the appropriate print function in `print.c` or add a new one
3. Connect the new format option to command-line arguments or `\pset` commands

### Adding Connection Parameters

1. Update option parsing in `parse_psql_options()` in `startup.c`
2. Pass the new parameter to connection functions

### Adding a Configuration Variable

1. Add the variable to `EstablishVariableSpace()` in `startup.c`
2. Create appropriate hook functions if variable changes affect runtime behavior

## Building and Testing

To build psql after making changes:

```bash
cd src/bin/psql
make
make install
```

For testing:
- Create regression tests in `src/bin/psql/t/`
- Run with `make check`
- Test manually with various connection scenarios and command types

## Debugging Tips

- Set verbosity with `\set VERBOSITY verbose`
- Enable command echoing with `\set ECHO all`
- Print variable values with `\echo :variable_name`
- Use `-v` command-line option for variable assignment

## Common Pitfalls

- Always check return values from libpq functions
- Free memory from PQresult with PQclear()
- Properly terminate strings in PQExpBuffer
- Handle special characters in SQL commands
- Remember to handle terminal encoding issues

## Additional Resources

- PostgreSQL source code documentation: https://doxygen.postgresql.org/
- PostgreSQL developer mailing lists: https://www.postgresql.org/list/
- libpq documentation: https://www.postgresql.org/docs/current/libpq.html
- Regression tests in `src/bin/psql/t/` provide examples of expected behavior
