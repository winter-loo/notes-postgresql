# PostgreSQL libpq Library Architecture

## Overview

The libpq library is PostgreSQL's official C client interface. It's a set of functions that allow client programs to pass queries to the PostgreSQL backend server and receive the results of these queries. libpq is also the underlying engine for several other PostgreSQL application interfaces, including those written for C++, Perl, Python, Tcl and ECPG.

## Main Components

The libpq library consists of several key components:

1. **Connection Management**: Functions to establish, maintain, and terminate connections to PostgreSQL servers
2. **Query Execution**: Functions to send queries and receive results
3. **Result Processing**: Functions to extract data from query results
4. **Asynchronous Processing**: Support for non-blocking operations
5. **Large Object Support**: Functions for handling large objects
6. **Error Handling**: Functions for error reporting and handling
7. **SSL/TLS Support**: Functions for secure connections
8. **COPY Protocol Support**: Functions for bulk data transfer

## Processing Flow

The typical flow of using libpq involves:

1. **Connection Establishment**: Create a connection to the server using connection parameters
2. **Query Execution**: Send SQL commands to the server
3. **Result Processing**: Retrieve and process the results
4. **Connection Termination**: Close the connection when done

## Key Data Structures

### PGconn

The `PGconn` structure represents a connection to the PostgreSQL server. It contains:

- Connection parameters (host, port, database, username, etc.)
- Connection status information
- Socket and protocol information
- Input/output buffers
- Result processing state
- Error messages
- SSL/TLS state (if applicable)

### PGresult

The `PGresult` structure represents the result of a query. It contains:

- Result status (success, error, etc.)
- Number of rows and columns
- Column descriptions (names, types, etc.)
- Row data
- Error messages (if applicable)
- Memory management information

### PQconninfoOption

This structure represents connection options that can be specified when establishing a connection.

## Design Aspects

### Connection Pooling

libpq does not provide built-in connection pooling. Applications requiring connection pooling must implement it themselves or use middleware like PgBouncer.

### Asynchronous Interface

libpq provides both synchronous and asynchronous interfaces:

- **Synchronous**: Functions like `PQexec()` that block until the operation completes
- **Asynchronous**: Functions like `PQsendQuery()` and `PQgetResult()` that allow non-blocking operation

### Pipeline Mode

In PostgreSQL 14 and later, libpq supports pipeline mode, which allows multiple queries to be sent to the server without waiting for the results of previous queries.

### Thread Safety

libpq is thread-safe if initialized properly, but each connection should be used by only one thread at a time.

### Error Handling

libpq provides detailed error information through the `PGresult` structure and connection error messages.

## Source Files and Their Roles

### Main Interface Files

- `libpq-fe.h`: Frontend interface definitions
- `fe-connect.c`: Connection establishment and management
- `fe-exec.c`: Query execution and result processing
- `fe-protocol3.c`: Protocol version 3 implementation
- `fe-auth.c`: Authentication methods
- `fe-secure.c`: SSL/TLS support

### Support Files

- `fe-misc.c`: Miscellaneous utility functions
- `fe-print.c`: Result printing functions
- `fe-lobj.c`: Large object support
- `pqexpbuffer.c`: Expandable string buffer implementation

## Protocol Implementation

libpq implements the PostgreSQL frontend/backend protocol, which is a message-based protocol with distinct phases:

1. **Connection Startup**:
   - Client sends a startup message with parameters (user, database, etc.)
   - Server responds with authentication request or ready for query

2. **Authentication**:
   - Multiple authentication methods supported (password, MD5, SCRAM, GSS, SSPI)
   - Challenge-response exchanges for secure methods
   - Implemented in `fe-auth.c` and related files

3. **Query Execution**:
   - Simple query protocol (single text command)
   - Extended query protocol (parse, bind, execute phases)
   - Parameter binding and result processing

4. **Data Transfer**:
   - Row description messages define result structure
   - Data row messages contain actual data
   - COPY protocol for bulk data transfer

## Memory Management

libpq employs sophisticated memory management techniques to efficiently handle query results:

1. **Block-based Allocation**:
   - Uses large blocks of memory instead of individual malloc calls
   - Reduces fragmentation and malloc/free overhead
   - Implemented in `fe-exec.c` for PGresult structures

2. **Result Object Lifecycle**:
   - Created by query execution functions
   - Populated with metadata and data from server responses
   - Must be explicitly freed by the application using PQclear()
   - Memory is tracked and managed within the result object

3. **Buffer Management**:
   - Dynamic input and output buffers that grow as needed
   - Efficient handling of large result sets
   - Optimized for both small and large messages

## Authentication Mechanisms

libpq supports multiple authentication methods:

1. **Password-based**:
   - Cleartext password (when SSL/TLS is used)
   - MD5 password hashing
   - SCRAM-SHA-256 authentication (RFC 7677)

2. **Kerberos/GSSAPI**:
   - Integration with system Kerberos infrastructure
   - Ticket exchange and validation
   - Optional delegation of credentials

3. **SSPI (Windows)**:
   - Windows-specific security provider interface
   - Supports both Kerberos and NTLM

4. **Certificate-based**:
   - Client certificate authentication with SSL/TLS
   - Certificate validation and verification

## Pipeline Mode

Pipeline mode is a feature introduced in PostgreSQL 14 that allows clients to send multiple queries to the server without waiting for the results of previous queries. This can significantly improve performance in high-latency environments.

### Implementation Details

1. **State Management**:
   - Pipeline state is tracked in the `PGconn` structure via `pipelineStatus` field
   - Three states: `PQ_PIPELINE_OFF`, `PQ_PIPELINE_ON`, and `PQ_PIPELINE_ABORTED`
   - Command queue tracks pending operations

2. **Command Queue**:
   - Each command is represented by a `PGcmdQueueEntry` structure
   - Queue entries track command type, status, and results
   - Queue is processed in FIFO order

3. **Synchronization Points**:
   - `PQpipelineSync()` creates a synchronization point
   - Server processes all previous commands and returns a sync response
   - Used to detect and recover from errors

4. **Error Handling**:
   - Errors don't immediately abort the pipeline
   - Processing continues until a sync point
   - Applications can detect errors and decide how to proceed

### Key Functions

- `PQenterPipelineMode()`: Switches a connection to pipeline mode
- `PQexitPipelineMode()`: Exits pipeline mode and returns to normal operation
- `PQpipelineSync()`: Establishes a synchronization point
- `PQsendFlushRequest()`: Requests server to flush its output buffer

## SSL/TLS Support

libpq provides comprehensive SSL/TLS support for secure client-server communications:

### Implementation Details

1. **Connection Encryption**:
   - Implemented in `fe-secure.c` and related files
   - Supports TLS protocol negotiation and version selection
   - Certificate validation with customizable verification levels

2. **Certificate Management**:
   - Client certificate authentication
   - Server certificate verification
   - Certificate revocation list (CRL) checking
   - Support for custom certificate stores

3. **Security Parameters**:
   - Cipher suite selection and configuration
   - Minimum TLS version enforcement
   - Perfect forward secrecy options
   - Channel binding for enhanced authentication security

4. **Integration with Authentication**:
   - SSL certificate authentication
   - Protection of password-based authentication
   - Channel binding for SCRAM authentication (SCRAM-SHA-256-PLUS)

### Connection Flow

1. Initial TCP connection established
2. SSL/TLS negotiation initiated with SSLRequest message
3. TLS handshake performed, certificates exchanged and validated
4. Encrypted channel established for all subsequent communication
5. Regular authentication proceeds over the encrypted channel

### Key Functions

- `pqsecure_open_client()`: Initiates SSL/TLS connection
- `pqsecure_read()` and `pqsecure_write()`: Handle encrypted I/O
- `pqsecure_close()`: Properly closes the secure connection
- `PQsslInUse()`: Checks if SSL/TLS is active on the connection

## Large Object Support

libpq provides a comprehensive API for working with PostgreSQL large objects, which are stored in a special large-object storage system within the database:

### Implementation Details

1. **Large Object API**:
   - Implemented in `fe-lobj.c`
   - Provides client-side functions for creating, opening, reading, writing, and deleting large objects
   - Supports seeking within large objects and truncating them
   - Handles importing/exporting between large objects and local files

2. **Internal Architecture**:
   - Large objects are identified by OIDs (Object Identifiers)
   - Client-side functions map to server-side large object functions via `lo_*` function calls
   - Uses file descriptor abstraction for open large objects
   - Supports both 32-bit and 64-bit file offsets (via separate function sets)

3. **Initialization Process**:
   - The `lo_initialize()` function dynamically discovers the OIDs of server-side large object functions
   - This allows compatibility across different PostgreSQL versions
   - Function OIDs are cached in the connection object for reuse

4. **Transaction Management**:
   - Large object operations must be enclosed in a transaction
   - The library doesn't automatically start transactions for large object operations

### Key Functions

- `lo_create()`: Creates a new large object
- `lo_open()`: Opens an existing large object for reading or writing
- `lo_write()`: Writes data to a large object
- `lo_read()`: Reads data from a large object
- `lo_lseek()` and `lo_lseek64()`: Positions the read/write pointer
- `lo_tell()` and `lo_tell64()`: Returns the current position
- `lo_unlink()`: Removes a large object
- `lo_import()` and `lo_export()`: Transfers data between files and large objects

## COPY Protocol Implementation

The COPY protocol is PostgreSQL's mechanism for efficient bulk data transfer between client and server:

### Implementation Details

1. **Protocol Modes**:
   - COPY FROM: Client sends data to server (COPY IN mode)
   - COPY TO: Server sends data to client (COPY OUT mode)
   - COPY BOTH: Bidirectional data transfer (introduced in newer versions)

2. **State Management**:
   - Connection state tracks COPY mode in `asyncStatus` field
   - Possible states: `PGASYNC_COPY_IN`, `PGASYNC_COPY_OUT`, and `PGASYNC_COPY_BOTH`
   - State transitions occur based on protocol messages and API calls

3. **Data Transfer**:
   - Data is sent in chunks using `CopyData` protocol messages
   - End of data is signaled with `CopyDone` or `CopyFail` messages
   - Streaming interface allows processing data without buffering entire dataset

4. **Error Handling**:
   - COPY operations can be aborted with error messages
   - Clean termination ensures protocol synchronization is maintained
   - Proper cleanup happens even when errors occur

### API Functions

1. **Modern Interface**:
   - `PQputCopyData()`: Sends a buffer of data to the server during COPY IN
   - `PQputCopyEnd()`: Signals end of COPY IN data, with optional error message
   - `PQgetCopyData()`: Receives data from the server during COPY OUT

2. **Legacy Interface** (deprecated):
   - `PQputline()`: Sends a text line during COPY IN
   - `PQgetline()`: Gets a text line during COPY OUT
   - `PQendcopy()`: Synchronizes at end of COPY

3. **Implementation**:
   - Implemented primarily in `fe-exec.c` and `fe-protocol3.c`
   - Integrates with the main protocol message processing system
   - Supports both synchronous and asynchronous operation

## Thread Safety and Concurrency

libpq is designed to be thread-safe, allowing multiple threads to use the library concurrently without conflicts:

### Thread Safety Design

1. **Connection Isolation**:
   - Each `PGconn` connection object is designed to be used by a single thread at a time
   - Different threads can safely use different connection objects concurrently
   - Applications must ensure that a single connection is not accessed simultaneously by multiple threads

2. **Thread Locking Mechanism**:
   - libpq provides a thread locking callback mechanism for operations that require thread synchronization
   - Implemented through the `PQregisterThreadLock()` function, which allows registering a custom thread locking handler
   - By default, uses a simple mutex-based implementation (`default_threadlock()` in `fe-connect.c`)

3. **Kerberos/GSSAPI Support**:
   - Special thread locking is primarily needed for Kerberos/GSSAPI authentication
   - The thread locking callback protects non-thread-safe functions used during authentication
   - Functions like `pglock_thread()` and `pgunlock_thread()` are used in authentication code

4. **SSL/TLS Implementation**:
   - SSL/TLS support includes thread-safety considerations
   - Uses mutex protection for shared SSL configuration operations
   - Signal handling (e.g., SIGPIPE masking) is handled in a thread-safe manner

### API Thread Safety

1. **Thread-Safe Functions**:
   - Most libpq functions are thread-safe when used with separate connection objects
   - The `PQisthreadsafe()` function returns whether libpq was built with thread safety (always returns true in modern versions)

2. **Non-Thread-Safe Functions**:
   - Some legacy functions are explicitly marked as non-thread-safe (e.g., old cancellation functions)
   - The documentation warns against using these in multi-threaded applications

3. **Windows Implementation**:
   - On Windows, libpq includes a partial pthread implementation (`pthread-win32.c`)
   - Provides mutex functions compatible with the POSIX threading API
   - Ensures consistent thread-safety behavior across platforms

### Best Practices

1. **Connection Per Thread**:
   - Use separate connection objects for each thread
   - If connection sharing is necessary, implement application-level synchronization

2. **Custom Thread Locking**:
   - Applications with special threading needs can provide custom locking handlers
   - Particularly useful for applications that also use Kerberos/GSSAPI directly

3. **Cancellation Handling**:
   - Use the thread-safe `PQcancel` functions rather than deprecated alternatives
   - Cancellation objects (`PGcancel`) can be safely used across threads

## Asynchronous Query Processing

libpq provides a comprehensive asynchronous API that allows applications to execute queries without blocking:

### Architecture Overview

1. **Non-Blocking Mode**:
   - Connections can be set to non-blocking mode with `PQsetnonblocking()`
   - In non-blocking mode, functions return immediately even if they cannot complete their operation
   - Applications must manage I/O readiness using select/poll/epoll or similar mechanisms

2. **State Machine**:
   - The connection object maintains state information about ongoing operations
   - The `asyncStatus` field tracks the current state (IDLE, BUSY, etc.)
   - State transitions occur based on protocol messages and API calls

3. **Command Queue**:
   - In pipeline mode, commands are queued for execution
   - Each command is represented by a `PGcmdQueueEntry` structure
   - The queue manages command execution order and result processing

### Key Components

1. **Query Submission**:
   - `PQsendQuery()`: Submits a simple query asynchronously
   - `PQsendQueryParams()`: Submits a parameterized query asynchronously
   - `PQsendPrepare()`: Prepares a statement asynchronously
   - `PQsendQueryPrepared()`: Executes a prepared statement asynchronously

2. **Result Processing**:
   - `PQconsumeInput()`: Reads available data from the server without blocking
   - `PQisBusy()`: Checks if the connection is busy processing input
   - `PQgetResult()`: Retrieves the next result from an asynchronous query
   - `PQsetSingleRowMode()`: Enables row-by-row result processing

3. **Internal Processing**:
   - `pqParseInput3()`: Parses incoming protocol messages
   - `pqPrepareAsyncResult()`: Prepares result objects for return to the application
   - `pqRowProcessor()`: Processes incoming row data
   - `pqCommandQueueAdvance()`: Advances the command queue after completing a command

### Implementation Details

1. **Result Building**:
   - Results are built incrementally as data arrives from the server
   - Memory management uses a block-based allocation strategy
   - Result objects (`PGresult`) contain metadata, row data, and error information

2. **Error Handling**:
   - Errors can occur at various stages (connection, query submission, result processing)
   - Error messages are stored in the connection object and/or result objects
   - Functions like `pqSaveErrorResult()` and `pqSetResultError()` manage error state

3. **Partial Result Modes**:
   - Single row mode allows processing large result sets without buffering all rows
   - Chunked rows mode provides batched result processing
   - These modes help manage memory usage for large result sets

### Asynchronous Operation Flow

1. **Submission Phase**:
   - Application calls an asynchronous query function
   - libpq formats and sends the appropriate protocol messages
   - Function returns immediately, indicating success or failure of submission

2. **Polling Phase**:
   - Application monitors connection for readiness (using select/poll/etc.)
   - When ready, application calls `PQconsumeInput()` to read available data
   - `PQisBusy()` determines if more input is expected

3. **Result Phase**:
   - Application calls `PQgetResult()` to retrieve results
   - Multiple calls may be needed to retrieve all results
   - NULL return indicates no more results for the current command

## Links to Key Source Files

- [libpq-fe.h](../postgresql/src/interfaces/libpq/libpq-fe.h)
- [libpq-int.h](../postgresql/src/interfaces/libpq/libpq-int.h)
- [fe-connect.c](../postgresql/src/interfaces/libpq/fe-connect.c)
- [fe-exec.c](../postgresql/src/interfaces/libpq/fe-exec.c)
- [fe-protocol3.c](../postgresql/src/interfaces/libpq/fe-protocol3.c)
- [fe-auth.c](../postgresql/src/interfaces/libpq/fe-auth.c)
- [fe-auth-scram.c](../postgresql/src/interfaces/libpq/fe-auth-scram.c)
- [fe-lobj.c](../postgresql/src/interfaces/libpq/fe-lobj.c)
- [pqexpbuffer.c](../postgresql/src/interfaces/libpq/pqexpbuffer.c)

## Links to Key Functions

- [PQconnectdb()](../postgresql/src/interfaces/libpq/fe-connect.c) - Establish a connection to a server
- [PQexec()](../postgresql/src/interfaces/libpq/fe-exec.c) - Execute a command synchronously
- [PQprepare()](../postgresql/src/interfaces/libpq/fe-exec.c) - Create a prepared statement
- [PQexecPrepared()](../postgresql/src/interfaces/libpq/fe-exec.c) - Execute a prepared statement
- [PQsendQuery()](../postgresql/src/interfaces/libpq/fe-exec.c) - Send a command asynchronously
- [PQgetResult()](../postgresql/src/interfaces/libpq/fe-exec.c) - Get the result of an asynchronous command
- [PQfinish()](../postgresql/src/interfaces/libpq/fe-connect.c) - Close a connection and free resources
- [PQreset()](../postgresql/src/interfaces/libpq/fe-connect.c) - Reset a connection
- [pqParseInput3()](../postgresql/src/interfaces/libpq/fe-protocol3.c) - Parse input from the server
