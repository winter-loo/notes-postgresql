# PostgreSQL LIMIT Operator Implementation

This document provides a detailed explanation of how the LIMIT operator is implemented in PostgreSQL, following the execution flow from parser to optimizer to executor.

## 1. Parser Implementation

The LIMIT clause in PostgreSQL is parsed in the grammar file (`src/backend/parser/gram.y`). The parser handles both the standard PostgreSQL syntax (`LIMIT count [OFFSET offset]`) and the SQL:2008 syntax (`FETCH FIRST count ROWS [ONLY|WITH TIES]`).

### Grammar Rules

The parser defines several grammar rules for the LIMIT clause:
- `limit_clause`: Handles the `LIMIT count` syntax
- `offset_clause`: Handles the `OFFSET count` syntax
- `fetch_clause`: Handles the SQL:2008 `FETCH FIRST count ROWS` syntax

During parsing, PostgreSQL creates a `SelectLimit` structure that contains:
- `limitCount`: The expression for the count parameter
- `limitOffset`: The expression for the offset parameter
- `limitOption`: The type of limit (COUNT or WITH TIES)

### LIMIT Options

PostgreSQL supports two limit options:
- `LIMIT_OPTION_COUNT`: Standard limit that returns exactly N rows (or fewer if the result set is smaller)
- `LIMIT_OPTION_WITH_TIES`: Returns N rows plus any additional rows that tie with the last row based on the ORDER BY clause

## 2. Optimizer Implementation

The optimizer processes the LIMIT clause in several phases:

### 2.1 Expression Preprocessing

In `preprocess_limit` (in `src/backend/optimizer/plan/planner.c`), PostgreSQL:
- Evaluates constant expressions in LIMIT/OFFSET clauses
- Estimates the number of rows that will be retrieved
- Passes this information to the query planner to help with cost estimation

### 2.2 Plan Creation

The LIMIT node is added to the query plan in `create_plan` after other operations like sorting, grouping, etc. The optimizer determines:
- Whether a LIMIT node is needed at all (skipped for constant-zero OFFSET and constant-null LIMIT)
- Where to place the LIMIT node in the plan tree

### 2.3 Optimization Opportunities

The optimizer uses LIMIT information to:
- Adjust `tuple_fraction` to help lower nodes understand how many rows are needed
- Potentially avoid full sorts when only a few rows are needed
- Create more efficient plans when LIMIT 1 is used (e.g., using index lookups instead of scans)

## 3. Executor Implementation

The executor implements LIMIT in `nodeLimit.c` using a state machine approach.

### 3.1 Data Structures

The `LimitState` structure maintains the state for a LIMIT node:
- `limitOffset`: Expression state for the OFFSET parameter
- `limitCount`: Expression state for the COUNT parameter
- `limitOption`: Type of limit (COUNT or WITH TIES)
- `offset`: Current OFFSET value
- `count`: Current COUNT value
- `noCount`: Flag indicating no count was specified
- `lstate`: Current state in the state machine
- `position`: 1-based index of the last tuple returned
- `subSlot`: Tuple last obtained from the subplan
- `eqfunction`: Tuple equality function for WITH TIES option
- `last_slot`: Slot for evaluating ties

### 3.2 State Machine

The LIMIT executor uses a state machine with the following states:
- `LIMIT_INITIAL`: Initial state before parameters are computed
- `LIMIT_RESCAN`: State after parameters are computed or during rescan
- `LIMIT_EMPTY`: No rows to return (either subplan is empty or count <= 0)
- `LIMIT_INWINDOW`: Currently returning rows within the LIMIT window
- `LIMIT_WINDOWEND_TIES`: Handling tied rows at the end of the window
- `LIMIT_SUBPLANEOF`: Reached end of subplan within the window
- `LIMIT_WINDOWEND`: Stepped off the end of the window
- `LIMIT_WINDOWSTART`: Stepped off the beginning of the window

### 3.3 Execution Flow

The main execution function `ExecLimit` follows this process:
1. On first call, compute the LIMIT/OFFSET values from expressions
2. Skip the first `offset` tuples from the subplan
3. Return up to `count` tuples from the subplan
4. If using WITH TIES, continue returning tuples that tie with the last one
5. Handle backward scans appropriately

### 3.4 WITH TIES Implementation

For `LIMIT ... WITH TIES`, the executor:
1. Saves the last tuple in the window in `last_slot`
2. Uses the `eqfunction` to compare subsequent tuples with the last one
3. Returns additional tuples that are equal according to the ORDER BY clause

## 4. Special Cases and Optimizations

### 4.1 LIMIT ALL

`LIMIT ALL` is represented as a NULL constant and effectively means no limit.

### 4.2 LIMIT with Backward Scans

PostgreSQL supports backward scans through LIMIT by maintaining position information and reversing the logic.

### 4.3 Parallel Execution

LIMIT nodes can complicate parallel execution since they require gathering a specific number of rows. The planner may decide not to use parallelism if it would be inefficient with a small LIMIT value.

## 5. Integration with Other Features

### 5.1 LIMIT with ORDER BY

LIMIT is often used with ORDER BY. The executor ensures that:
- Sorting happens before limiting
- WITH TIES option works correctly with the ORDER BY keys

### 5.2 LIMIT in Subqueries

LIMIT in subqueries is handled differently depending on the context:
- In EXISTS subqueries, LIMIT 1 is an optimization that doesn't change semantics
- In set-returning subqueries, LIMIT affects the number of rows returned

## Conclusion

PostgreSQL's LIMIT implementation is a sophisticated system that spans the parser, optimizer, and executor components. The state machine approach in the executor provides flexibility for handling different scenarios, including forward and backward scans, while the optimizer uses LIMIT information to create more efficient query plans.
