# PostgreSQL Optimizer Architecture

This document describes the architecture of the PostgreSQL optimizer module, which is responsible for transforming parsed queries into efficient execution plans.

## Overview

The PostgreSQL optimizer (also called the planner) takes a parsed query tree (produced by the parser) and determines the most efficient way to execute the query. This involves analyzing various possible execution strategies and selecting the one with the lowest estimated cost.

The optimizer is located in the `src/backend/optimizer/` directory and is organized into several subdirectories:

- `plan/`: Contains code for generating the final execution plan
- `path/`: Contains code for generating and evaluating possible access paths
- `prep/`: Contains preprocessing steps for special cases
- `util/`: Contains utility functions used by the optimizer
- `geqo/`: Contains the Genetic Query Optimizer for handling queries with many joins

## Main Components and Processing Flow

### 1. Entry Point

The main entry point to the optimizer is the `planner()` function in `src/backend/optimizer/plan/planner.c`. This function:

- Takes a parsed Query structure from the parser
- Calls `standard_planner()` to perform the actual planning
- Returns a `PlannedStmt` structure containing the execution plan

```c
PlannedStmt *planner(Query *parse, const char *query_string, int cursorOptions,
                    ParamListInfo boundParams)
```

### 2. Standard Planner

The `standard_planner()` function in `planner.c` is the core planning function that:

- Initializes planning data structures
- Calls `subquery_planner()` to plan the main query and any subqueries
- Applies top-level processing like limit handling
- Converts the optimal path into a Plan tree

```c
PlannedStmt *standard_planner(Query *parse, const char *query_string, int cursorOptions,
                             ParamListInfo boundParams)
```

### 3. Subquery Planner

The `subquery_planner()` function in `planner.c` performs most of the planning work:

- Preprocesses the query (subqueries, quals, etc.)
- Calls `query_planner()` to generate paths for the FROM list
- Calls `grouping_planner()` to handle GROUP BY, aggregates, etc.

```c
PlannerInfo *subquery_planner(PlannerGlobal *glob, Query *parse, PlannerInfo *parent_root,
                             bool hasRecursion, double tuple_fraction,
                             SetOperationStmt *setops)
```

### 4. Query Planner

The `query_planner()` function in `src/backend/optimizer/path/allpaths.c` is responsible for:

- Building RelOptInfo structures for each base relation
- Setting up size estimates for base relations
- Generating access paths for base relations
- Building the join tree and finding the optimal join order

```c
RelOptInfo *query_planner(PlannerInfo *root, List *tlist, Query *parse)
```

### 5. Path Generation

Path generation is handled by various functions in the `path/` directory:

- `set_base_rel_sizes()`: Sets size estimates for base relations
- `set_base_rel_pathlists()`: Generates access paths for base relations
- `make_one_rel()`: Builds the join tree and finds the optimal join order

### 6. Join Planning

Join planning is one of the most complex parts of the optimizer:

- `make_rel_from_joinlist()`: Converts the join list into a RelOptInfo
- `standard_join_search()`: Performs the dynamic programming algorithm to find the optimal join order
- For queries with many joins, the genetic query optimizer (GEQO) is used instead

### 7. Grouping Planner

The `grouping_planner()` function in `planner.c` handles higher-level operations:

- GROUP BY processing
- Aggregate function processing
- DISTINCT processing
- Window function processing
- ORDER BY processing
- LIMIT/OFFSET processing

```c
void grouping_planner(PlannerInfo *root, double tuple_fraction,
                     SetOperationStmt *setops)
```

### 8. Plan Creation

Finally, the selected path is converted into a Plan tree by functions in `createplan.c`:

- `create_plan()`: Converts a Path to a Plan
- Various specialized functions for different plan types (e.g., `create_seqscan_plan()`, `create_indexscan_plan()`, etc.)

## Key Data Structures

### 1. PlannerInfo (root)

The central data structure that holds all planning information for a query level:

- Query being planned
- RelOptInfos for all relations
- Path creation contexts
- Planning state and statistics

### 2. RelOptInfo

Represents a relation (base table or join) during planning:

- Size estimates (rows, width)
- List of possible access paths
- Restriction and join clauses
- Parameterization information

### 3. Path

Represents a possible way to scan or join relations:

- Type of path (sequential scan, index scan, nested loop, etc.)
- Cost estimates (startup and total)
- Output ordering (pathkeys)
- Parameterization requirements

### 4. RestrictInfo

Represents a WHERE or JOIN clause:

- The clause expression
- Selectivity estimate
- Evaluation placement information

## Key Design Aspects

### 1. Cost-Based Optimization

PostgreSQL uses a cost-based optimizer that estimates the execution cost of different plans and chooses the one with the lowest cost. Costs are calculated based on:

- I/O costs (disk reads)
- CPU costs (processing time)
- Size estimates (number of rows, tuple width)

### 2. Dynamic Programming for Join Ordering

For queries with a small to moderate number of joins, PostgreSQL uses a dynamic programming algorithm to find the optimal join order:

- First considers joins between pairs of base relations
- Then builds up to larger join trees
- Considers left-deep, right-deep, and bushy join trees

### 3. Genetic Algorithm for Complex Queries

For queries with many joins (more than `geqo_threshold`, which defaults to 12), PostgreSQL uses a genetic algorithm to find a good (but not necessarily optimal) join order:

- Represents join orders as "chromosomes"
- Uses selection, crossover, and mutation to evolve better solutions
- Trades optimality for planning speed

### 4. Parameterized Paths

PostgreSQL supports parameterized paths, where a scan or join depends on values from outer relations:

- Allows pushing join conditions down into inner scans
- Can significantly improve performance for nested loop joins

### 5. Equivalence Classes

The optimizer tracks sets of expressions known to be equal (equivalence classes):

- Used for redundant join clause elimination
- Helps determine which orderings are available "for free"
- Enables more efficient plan selection

## Source Files and Their Roles

### plan/ directory

- `planner.c`: Main entry point and high-level planning
- `createplan.c`: Converts Paths to Plans
- `subselect.c`: Handles subqueries
- `planmain.c`: Plan finalization

### path/ directory

- `allpaths.c`: Path generation for all relation types
- `joinpath.c`: Path generation for joins
- `pathkeys.c`: Management of ordering information
- `costsize.c`: Cost and size estimation
- `indxpath.c`: Index path generation
- `equivclass.c`: Equivalence class management

### prep/ directory

- `prepqual.c`: Preprocessing of WHERE clauses
- `preptlist.c`: Preprocessing of target lists
- `prepunion.c`: Preprocessing of UNION/INTERSECT/EXCEPT
- `prepjointree.c`: Preprocessing of join trees

### geqo/ directory

- `geqo_main.c`: Main genetic optimizer code
- `geqo_selection.c`, `geqo_mutation.c`, etc.: Genetic algorithm components

### util/ directory

- `clauses.c`: Clause manipulation and evaluation
- `predtest.c`: Predicate testing
- `relnode.c`: RelOptInfo management
- `pathnode.c`: Path node creation and management

## Links to Key Source Files

- [planner.c](../postgresql/src/backend/optimizer/plan/planner.c)
- [allpaths.c](../postgresql/src/backend/optimizer/path/allpaths.c)
- [joinpath.c](../postgresql/src/backend/optimizer/path/joinpath.c)
- [costsize.c](../postgresql/src/backend/optimizer/path/costsize.c)
- [geqo_main.c](../postgresql/src/backend/optimizer/geqo/geqo_main.c)

## Links to Key Functions

- [planner()](../postgresql/src/backend/optimizer/plan/planner.c)
- [standard_planner()](/home/ldd/docs-postgresql/postgresql/src/backend/optimizer/plan/planner.c)
- [subquery_planner()](../postgresql/src/backend/optimizer/plan/planner.c)
- [query_planner()](../postgresql/src/backend/optimizer/path/allpaths.c)
- [make_one_rel()](../postgresql/src/backend/optimizer/path/allpaths.c)
- [standard_join_search()](../postgresql/src/backend/optimizer/path/joinpath.c)
- [grouping_planner()](../postgresql/src/backend/optimizer/plan/planner.c)
