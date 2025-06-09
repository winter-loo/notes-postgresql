# SQL

```sql
declare c4 cursor for select * from int8_tbl;
```

# Challenges

1. **Query Planning Without Execution**: Implementing a mechanism to parse and plan the query without immediate execution, while ensuring the plan remains valid when the cursor is later used.

2. **Memory Management**: Balancing memory usage for potentially large result sets, especially when multiple cursors are active simultaneously.

3. **Transaction Isolation**: Ensuring proper snapshot isolation so that the cursor's view of the data remains consistent throughout its lifetime, regardless of concurrent modifications.

4. **Cursor Lifecycle Management**: Tracking cursor state across multiple fetch operations while properly handling transaction boundaries (commit/rollback).

5. **Performance Optimization**: Implementing efficient fetching mechanisms that don't require materializing the entire result set upfront.

6. **Cursor Options Handling**: Supporting various cursor options (SCROLL/NO SCROLL, WITH HOLD/WITHOUT HOLD, etc.) that affect cursor behavior.

7. **Error Handling**: Properly managing errors that might occur during cursor declaration versus errors during fetch operations.

8. **Resource Cleanup**: Ensuring proper cleanup of resources when cursors are explicitly closed or implicitly closed at transaction end.

9. **Concurrency Control**: Managing potential conflicts between cursor operations and concurrent DDL operations on the underlying tables.

# Solution

PostgreSQL implements the following solutions to address these challenges:

1. **Query Planning Without Execution**:
   - Uses the [`PerformCursorOpen`](../../postgresql/src/backend/commands/portalcmds.c#L42) function in [`portalcmds.c`](../../postgresql/src/backend/commands/portalcmds.c) to parse and plan the query
   - Stores the planned query in a Portal structure without executing it
   - The query plan is created with [`pg_plan_query`](../../postgresql/src/backend/tcop/pquery.c) and stored in the portal's memory context
   - Execution is deferred until `FETCH` or `MOVE` commands are issued

2. **Memory Management**:
   - Implements a dedicated [`PortalContext`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L166) memory context for each cursor
   - Uses [`portalmem.c`](../../postgresql/src/backend/utils/mmgr/portalmem.c) to manage portal memory allocation and cleanup
   - For holdable cursors, materializes results in a tuplestore to avoid keeping executor state alive
   - Non-holdable cursors maintain minimal state between fetch operations

3. **Transaction Isolation**:
   - Captures the active snapshot at cursor creation time with [`GetActiveSnapshot()`](../../postgresql/src/backend/utils/time/snapmgr.c)
   - Preserves this snapshot for the cursor's lifetime to ensure consistent data view
   - For holdable cursors, materializes results before transaction end to maintain consistency

4. **Cursor Lifecycle Management**:
   - Implements portal state transitions (READY → ACTIVE → DONE/FAILED) in [`portalmem.c`](../../postgresql/src/backend/utils/mmgr/portalmem.c)
   - Tracks cursor state using a hash table ([`PortalHashTable`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L39)) for fast lookup
   - Handles transaction boundaries with functions like [`PreCommit_Portals`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L663), [`AtAbort_Portals`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L773)
   - Special handling for holdable cursors via [`HoldPortal`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L631) and [`PersistHoldablePortal`](../../postgresql/src/backend/commands/portalcmds.c#L316)

5. **Performance Optimization**:
   - Implements demand-driven execution through the executor interface
   - Fetches rows only as needed rather than materializing the entire result set
   - For scrollable cursors, uses executor's backward scan capability when available
   - Automatically determines if a cursor can be scrollable based on the plan's capabilities

6. **Cursor Options Handling**:
   - Stores cursor options in [`portal->cursorOptions`](../../postgresql/src/include/utils/portal.h) bitmask
   - Validates options at declaration time (e.g., WITH HOLD not allowed in security contexts)
   - Automatically determines scrollability based on query plan characteristics
   - Enforces transaction requirements (non-holdable cursors require transaction block)

7. **Error Handling**:
   - Uses [`MarkPortalFailed`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L434) to mark portals that encounter errors during execution
   - Implements cleanup handlers for both normal and error conditions
   - Provides detailed error messages for common cursor-related errors
   - Special handling for errors during holdable cursor materialization

8. **Resource Cleanup**:
   - Implements automatic cleanup at transaction boundaries
   - Provides explicit `CLOSE` command via [`PerformPortalClose`](../../postgresql/src/backend/commands/portalcmds.c#L218)
   - Uses resource owner mechanism to track and release cursor resources
   - Special handling for subtransaction abort via [`AtSubAbort_Portals`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L968) and [`AtSubCleanup_Portals`](../../postgresql/src/backend/utils/mmgr/portalmem.c#L1084)

9. **Concurrency Control**:
   - Uses the transaction's snapshot isolation to handle concurrent data modifications
   - For DDL conflicts, relies on PostgreSQL's locking system
   - Holdable cursors materialize results to avoid conflicts with schema changes
   - Non-holdable cursors are tied to transaction lifetime, limiting DDL conflict window
