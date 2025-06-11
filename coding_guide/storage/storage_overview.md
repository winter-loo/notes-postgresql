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
