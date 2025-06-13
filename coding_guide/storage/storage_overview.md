# PostgreSQL Storage Layer Overview

## Introduction

PostgreSQL's storage layer is responsible for persisting and retrieving data efficiently while ensuring durability, concurrency, and transactional integrity. This guide provides an overview of the key components and concepts of PostgreSQL's storage architecture.

## Storage Hierarchy

PostgreSQL's storage is organized in a hierarchical structure:

```
Cluster
  ├── Tablespaces
  │    ├── Databases
  │    │    ├── Relations (Tables, Indexes)
  │    │    │    ├── Files
  │    │    │    └── TOAST tables
  │    │    └── System catalogs
  │    └── Global objects
  └── WAL files
```

1. **Cluster**: A PostgreSQL instance manages a single cluster, which is a collection of databases.
2. **Tablespaces**: Physical locations where data files are stored. Default tablespaces include `pg_default` and `pg_global`.
3. **Databases**: Isolated collections of relations and objects.
4. **Relations**: Tables, indexes, sequences, and other database objects.
5. **Files**: Physical files on disk that store the actual data.

## Physical Storage

### File Organization

PostgreSQL stores each relation as one or more files in the file system. By default:

- Each relation larger than 1GB is split into multiple 1GB segment files.
- Files are named with the object identifier (OID) of the relation.
- For tables: `base/dboid/relfilenode`
- For indexes: `base/dboid/relfilenode`
- TOAST tables: `base/dboid/relfilenode_toast`

### Page Structure

PostgreSQL divides each file into fixed-size pages (blocks), typically 8KB:

```
+------------------+
| Page Header      | 24 bytes
+------------------+
| Item Pointers    | Variable size
+------------------+
| Free Space       | Variable size
+------------------+
| Items (Tuples)   | Variable size
+------------------+
| Special Space    | Variable size (index-specific data)
+------------------+
```

- **Page Header**: Contains metadata like checksum, LSN, and free space pointers.
- **Item Pointers**: Array of (offset, length) pairs pointing to tuples.
- **Free Space**: Area available for new tuples.
- **Items**: The actual tuple data.
- **Special Space**: Used primarily by indexes for special metadata.

### Tuple Structure

Tuples (rows) have a specific format:

```
+------------------+
| Header           | (23 bytes + bitmap)
|  - t_xmin        | (Creating transaction ID)
|  - t_xmax        | (Deleting transaction ID)
|  - t_cid         | (Command ID)
|  - t_ctid        | (Current TID - for updates)
|  - t_infomask    | (Flags)
|  - t_hoff        | (Header size)
|  - t_bits        | (Null bitmap)
+------------------+
| Data             | (Actual column values)
+------------------+
```

- Transaction IDs (t_xmin/t_xmax) are used for MVCC (Multi-Version Concurrency Control)
- The current TID (t_ctid) points to the newest version of the tuple for updates
- The null bitmap tracks which attributes are NULL

## TOAST (The Oversized-Attribute Storage Technique)

TOAST is used for storing large field values:

1. **Compression**: Large values are compressed when possible.
2. **Out-of-line storage**: Very large values are stored in separate TOAST tables.
3. **Chunking**: Large values are split into chunks of ~2KB each.

TOAST allows PostgreSQL to efficiently handle large field values while maintaining good performance for normal operations.

## Buffer Management

The buffer manager mediates between disk and memory:

1. **Shared Buffers**: A cache of recently used data pages in shared memory.
2. **Buffer Tags**: Unique identifiers for cached pages (relfilenode, fork, block number).
3. **Replacement Policy**: Clock-sweep algorithm to decide which pages to evict when the cache is full.
4. **Ring Buffer**: WAL buffers that store transaction logs before they're written to disk.

The buffer manager ensures efficient I/O operations by:
- Reading pages only once
- Batching writes
- Supporting different I/O patterns (sequential scan, random access)

## WAL (Write-Ahead Logging)

The Write-Ahead Logging (WAL) subsystem is a critical component of PostgreSQL's architecture, ensuring data integrity and durability. This developer guide aims to provide a comprehensive overview of WAL's architecture, components, interfaces, and best practices for PostgreSQL developers.

### 1. Introduction

WAL implements the fundamental principle that changes to data files should only be written to disk after those changes have been logged to durable storage. This ensures that if a crash occurs, the database can recover to a consistent state by replaying the WAL records.

Key benefits of WAL include:
- Crash recovery (REDO) capability
- Support for point-in-time recovery
- Enabling replication and streaming replication
- Reducing physical I/O by batching disk writes

### 2. WAL Architecture

#### 2.1 WAL Files and Segments

WAL data is stored in files called segments. Each segment has a fixed size (default 16MB, configurable via `wal_segment_size`). Segments are named sequentially and organized in the `pg_wal` directory (formerly `pg_xlog` in older versions).

Format: `XXXXXXXXYYYYYYYY` where:
- `XXXXXXXX` is the timeline ID (hexadecimal)
- `YYYYYYYY` is the log segment number (hexadecimal)

#### 2.2 WAL Records

WAL records are variable-length entries containing:
- Header: Record length, CRC, previous record's location, etc.
- Resource Manager ID: Identifies which subsystem created the record
- Data: The actual change data specific to the record type

Records are immutable once written and are identified by their Log Sequence Number (LSN), which is a 64-bit value representing the byte position in the WAL stream.

#### 2.3 WAL Buffers

Changes are first written to in-memory WAL buffers before being flushed to disk. The number and size of these buffers are controlled by the `wal_buffers` parameter.

#### 2.4 Resource Managers

Each PostgreSQL subsystem that needs to record changes in WAL registers a Resource Manager that provides:
- `rm_info`: Record type information
- `rm_desc`: Functions to describe record contents
- `rm_redo`: Functions to replay changes
- `rm_undo`: Functions for rollback (optional)

### 3. WAL Workflow

#### 3.1 WAL Insertion

```c
XLogRecPtr XLogInsertRecord(XLogRecData *rdata, XLogRecPtr fpw_lsn, uint8 flags,
                           int num_fpi, bool topxid_included);
```

Typical usage pattern:
1. Prepare record data in an `XLogRecData` structure
2. Call `XLogInsertRecord()` to insert the record
3. Receive the LSN of the record as the return value

#### 3.2 WAL Flushing

```c
void XLogFlush(XLogRecPtr record);
```

Ensures all WAL data up to the specified LSN is written and fsync'd to disk. Critical for ensuring durability guarantees.

#### 3.3 Checkpoints

Checkpoints ensure all dirty data buffers are flushed to disk, limiting recovery time after a crash. They create a known-good point in the transaction log.

Triggered by:
- Time-based intervals (`checkpoint_timeout`)
- WAL volume (`max_wal_size`)
- Explicit command (`CHECKPOINT`)

### 4. WAL Record Types

WAL records are categorized by their Resource Manager and record type. Common types include:

- XLOG: Control information like checkpoints
- HEAP: Table data modifications
- BTREE: B-tree index modifications
- HASH: Hash index modifications
- GIN/GIST/SP-GIST: Other index types
- CLOG: Transaction commit status
- MULTIXACT: Multiple transaction information
- XACT: Transaction commit/abort records

### 5. Implementing WAL Support in Modules

To add WAL support to a PostgreSQL module:

1. Define record types and structures
2. Implement a Resource Manager
3. Add record creation logic during data modifications
4. Implement REDO functions for recovery

Example structure for a resource manager:

```c
static const RmgrData MyRmgrData = {
    .rm_name = "MyExtension",
    .rm_redo = my_redo,
    .rm_desc = my_desc,
    .rm_startup = NULL,
    .rm_cleanup = NULL,
    .rm_mask = NULL
};
```

#### 5.1 Custom WAL Resource Managers

Extensions can register custom WAL resource managers using the provided API:

```c
void RegisterCustomRmgr(uint8 rmid, const RmgrData *rmgr);
```

### 6. Generic WAL Records

For simpler cases, PostgreSQL provides a Generic WAL API that abstracts the details of WAL record creation:

```c
GenericXLogStart(Relation relation) → GenericXLogState *
GenericXLogRegisterBuffer(GenericXLogState *state, Buffer buffer, int flags)
GenericXLogFinish(GenericXLogState *state) → XLogRecPtr
```

This simplifies adding WAL support to extensions without implementing a full Resource Manager.

### 7. Asynchronous Commit

PostgreSQL offers different commit modes:

- `synchronous_commit = on`: Wait until WAL is flushed to disk (default)
- `synchronous_commit = off`: Return without waiting for WAL flush
- Other modes: (`remote_apply`, `remote_write`, `local`, etc.)

Developers should understand these modes' implications for data durability when implementing features.

### 8. WAL and Replication

WAL enables PostgreSQL's replication features:

- Streaming replication uses WAL to continuously send changes to standby servers
- Physical replication slots prevent needed WAL segments from being removed
- Logical replication decodes WAL into logical change sets

### 9. Key WAL-Related Data Structures

- `XLogRecPtr`: 64-bit LSN value identifying WAL positions
- `XLogRecord`: WAL record header structure
- `XLogRecData`: Structure used to build WAL records
- `XLogReaderState`: Used for reading and decoding WAL records

### 10. Best Practices

#### 10.1 Performance Considerations

- Group related changes into a single WAL record when possible
- Be mindful of WAL volume generated by operations
- Properly size WAL buffers for workload requirements
- Consider full-page writes impact for frequently updated pages

#### 10.2 Recovery Considerations

- Ensure REDO functions handle all possible edge cases
- Test recovery thoroughly with various failure scenarios
- Remember that WAL replay happens without transaction context

#### 10.3 Common Pitfalls

- Missing WAL records for required operations
- Incorrect ordering of WAL operations
- Insufficient data in WAL records for proper replay
- Forgetting to handle WAL during buffer modifications

### 11. Debugging and Monitoring

- `pg_waldump`: Tool to inspect WAL segment contents
- `pg_current_wal_lsn()`: SQL function to get current WAL position
- `pg_current_wal_insert_lsn()`: Get current insert position
- `pg_wal_replay_pause()` and `pg_wal_replay_resume()`: Control replay on standby
- `pg_stat_wal`: View for WAL statistics

### 12. Further Reading

- PostgreSQL source: `src/backend/access/transam/xlog.c` and related files
- PostgreSQL documentation: "Write-Ahead Log" chapter
- PostgreSQL wiki: "WAL" and "Custom WAL Resource Managers"

WAL ensures durability and crash recovery:

1. **Log Records**: Each change is first recorded in the WAL before modifying the actual data pages.
2. **LSN (Log Sequence Number)**: A pointer to a position in the WAL stream.
3. **Checkpoints**: Points where all modified buffers are guaranteed to be written to disk.
4. **Archive Mode**: WAL files can be archived for point-in-time recovery.

WAL is critical for:
- Crash recovery (REDO)
- Replication
- Point-in-time recovery

## Tablespaces

Tablespaces allow data to be stored in different physical locations:

1. **Default Tablespaces**: `pg_default` (user data) and `pg_global` (system catalogs).
2. **Custom Tablespaces**: Can be created on different disk volumes for performance or capacity reasons.

## Free Space Map (FSM) and Visibility Map (VM)

1. **Free Space Map**: Tracks available space in pages for efficient tuple insertion.
2. **Visibility Map**: Tracks which pages contain only tuples that are visible to all active transactions.

These auxiliary data structures improve performance by:
- Reducing I/O operations during tuple insertion
- Optimizing vacuum operations
- Enabling index-only scans

## VACUUM and Autovacuum

VACUUM is critical for PostgreSQL's MVCC model:

1. **VACUUM**: Reclaims space from dead tuples, updates the free space map, and freezes old transaction IDs.
2. **Autovacuum**: Automatically runs VACUUM and ANALYZE based on configured thresholds.
3. **VACUUM FULL**: Rewrites the entire table to reclaim space, requiring an exclusive lock.

## Indexes

PostgreSQL supports multiple index types:

1. **B-tree**: Default index type, suitable for equality and range queries.
2. **Hash**: Optimized for equality comparisons.
3. **GiST**: Generalized Search Tree for geometric data, full-text search.
4. **SP-GiST**: Space-partitioned GiST for non-balanced data structures.
5. **GIN**: Generalized Inverted Index for composite values.
6. **BRIN**: Block Range Index for large tables with natural ordering.

Index structure generally includes:
- Header pages with metadata
- Internal pages with pointers
- Leaf pages with actual index entries and references to heap tuples

## Transaction Management

Storage-level transaction management features:

1. **MVCC**: Provides transaction isolation without read locks.
2. **Transaction IDs**: 32-bit integers that wrap around (requiring periodic cleanup).
3. **Tuple visibility**: Determined by comparing transaction IDs with the current transaction.
4. **Commit Log (CLOG/pg_xact)**: Tracks transaction statuses (committed, aborted, in progress).

## System Catalogs

System catalogs store metadata about the database:

1. **pg_class**: Information about relations (tables, indexes, etc.)
2. **pg_attribute**: Information about columns of relations
3. **pg_type**: Information about data types
4. **pg_index**: Information about indexes
5. **pg_am**: Information about access methods

These catalogs are themselves regular tables with a special format.

## Conclusion

PostgreSQL's storage layer is complex but well-designed, offering:

- Durability through WAL
- Concurrency through MVCC
- Flexibility through pluggable index types
- Efficiency through buffer management and specialized storage techniques

Understanding these components helps in optimizing PostgreSQL for specific workloads and troubleshooting performance issues.
