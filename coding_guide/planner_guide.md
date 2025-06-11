# PostgreSQL SQL Optimizer Modification Guide

## Overview

The PostgreSQL optimizer (also known as the query planner) is responsible for transforming parsed and analyzed queries into efficient execution plans. This guide provides essential information for developers who want to understand and modify the PostgreSQL optimizer.

## Architecture

The PostgreSQL optimizer follows a modular architecture with several key components:

- **Path Generator**: Creates possible execution paths for scanning relations and performing joins
- **Cost Estimator**: Evaluates the cost of different execution paths
- **Plan Builder**: Transforms the selected paths into executable plans
- **GEQO (Genetic Query Optimizer)**: Handles queries with many joins using genetic algorithms

The optimization process follows these steps:
1. Generate access paths for base relations (tables, views, subqueries)
2. Build join paths by considering different join orders and methods
3. Apply optimization rules (predicate pushdown, join reordering, etc.)
4. Estimate costs for each path based on statistics and heuristics
5. Select the cheapest path (or cheapest path with desired ordering)
6. Convert the selected path tree into an executable plan tree

### Optimization Rule: Predicate Pushdown

Predicate pushdown is a crucial query optimization technique that moves filter conditions (predicates) as close as possible to the data sources. This reduces the amount of data processed in intermediate steps.

#### How Predicate Pushdown Works

1. **Basic Principle**: Filter data as early as possible in the execution plan
2. **Benefits**: Reduces I/O, memory usage, and CPU processing time
3. **Examples**: Pushing WHERE clauses into subqueries, views, or even down to storage engines

#### Implementation in PostgreSQL

In PostgreSQL, predicate pushdown is primarily implemented in `src/backend/optimizer/prep/prepqual.c` and `src/backend/optimizer/util/clauses.c`. Here's how it works:

```c
/* Function that tries to push down qualifications from a parent relation into subqueries */
static Query *
subquery_push_qual(Query *subquery, RangeTblEntry *rte, Index rti, Node *qual)
{
    /* Make a copy of the subquery to modify */
    Query *result = copyObject(subquery);

    /*
     * If the subquery has a LIMIT clause, we cannot push the qual into it
     * since that would change the number of rows returned.
     */
    if (result->limitCount)
        return result;

    /* Merge the qual into subquery's WHERE clause */
    if (result->jointree->quals)
        result->jointree->quals = make_and_qual(result->jointree->quals,
                                             copyObject(qual));
    else
        result->jointree->quals = copyObject(qual);

    /* The pushed-down quals may contain SubLinks, so update hasSubLinks */
    if (!result->hasSubLinks)
        result->hasSubLinks = checkExprHasSubLink(qual);

    return result;
}
```

#### Common Use Cases

1. **Simple Predicate Pushdown**: Consider a query with a WHERE clause on a table:
   ```sql
   SELECT * FROM orders JOIN order_items ON orders.id = order_items.order_id 
   WHERE orders.date > '2023-01-01';
   ```
   
   PostgreSQL will push the date predicate down to the `orders` table scan, filtering rows before the join operation.

2. **Subquery Predicate Pushdown**: For a query with a subquery:
   ```sql
   SELECT * FROM (
       SELECT * FROM products
   ) AS p WHERE p.price < 100;
   ```
   PostgreSQL will push the price filter inside the subquery evaluation.

#### Implementation Steps for Adding a New Pushdown Rule

1. Identify where predicates can be safely pushed down
2. Modify `prepqual.c` functions like `pull_up_sublinks()` and `pull_up_subqueries()`
3. Add logic to recognize and transform the specific predicate type
4. Ensure semantic equivalence before and after transformation
5. Add statistics collection for the optimization

#### Debugging Tips

1. Use `debug_print_qual` GUC to see quals as they're processed
2. Check `src/backend/optimizer/README` for optimizer internals
3. The function `distribute_restrictinfo_to_rels()` in `src/backend/optimizer/prep/prepjointree.c` is key for how quals get distributed between relations

## Key Files and Their Purposes

The optimizer code is divided into several subdirectories:

| Directory | Purpose |
|-----------|---------|
| `optimizer/path` | Creates and evaluates access paths |
| `optimizer/plan` | Converts paths to executable plans |
| `optimizer/prep` | Preprocesses queries before optimization |
| `optimizer/util` | Utility functions used across the optimizer |
| `optimizer/geqo` | Genetic query optimization for complex joins |

Key files include:

| File | Location | Description |
|------|----------|-------------|
| `allpaths.c` | `src/backend/optimizer/path` | Main entry point for path generation |
| `costsize.c` | `src/backend/optimizer/path` | Cost estimation functions |
| `joinpath.c` | `src/backend/optimizer/path` | Join path creation |
| `joinrels.c` | `src/backend/optimizer/path` | Join relation management |
| `pathkeys.c` | `src/backend/optimizer/path` | Sort ordering handling |
| `indxpath.c` | `src/backend/optimizer/path` | Index path creation |
| `createplan.c` | `src/backend/optimizer/plan` | Path to Plan conversion |
| `planner.c` | `src/backend/optimizer/plan` | Main planner entry point |
| `subselect.c` | `src/backend/optimizer/prep` | Subquery processing |

## Core Concepts and Data Structures

### RelOptInfo

`RelOptInfo` represents a base relation or join relation during planning:

```c
typedef struct RelOptInfo
{
    NodeTag     type;           /* identifies this as a RelOptInfo */
    RelOptKind  reloptkind;     /* RELOPT_BASEREL, RELOPT_JOINREL, etc */
    
    /* estimated size and cost information */
    double      rows;           /* estimated number of result tuples */
    Cost        startup_cost;   /* cost before first tuple returned */
    Cost        total_cost;     /* total cost (includes startup_cost) */
    
    List       *pathlist;       /* Path structures for this relation */
    Path       *cheapest_startup_path; /* cheapest path for retrieving first few tuples */
    Path       *cheapest_total_path;   /* cheapest path overall */
    Path       *cheapest_unique_path;  /* cheapest path for retrieving unique tuples */
    
    /* other important fields... */
} RelOptInfo;
```

### Path

`Path` represents a way to generate the tuples of a relation:

```c
typedef struct Path
{
    NodeTag     type;           /* tag identifying path node type */
    PathType    pathtype;       /* path node type (PATHTYPE_SEQSCAN etc) */
    RelOptInfo *parent;         /* relation this path can access */
    PathTarget *pathtarget;     /* target to be computed by this path */
    
    /* cost and size estimates */
    double      rows;           /* estimated number of result tuples */
    Cost        startup_cost;   /* cost before first tuple retrieved */
    Cost        total_cost;     /* total cost (includes startup_cost) */
    
    List       *pathkeys;       /* sort ordering of path's output */
    /* other fields... */
} Path;
```

### Plan

`Plan` represents an executable query plan:

```c
typedef struct Plan
{
    NodeTag     type;           /* tag identifying plan node type */
    
    Cost        startup_cost;   /* cost before first tuple returned */
    Cost        total_cost;     /* total cost (includes startup_cost) */
    
    double      plan_rows;      /* estimated number of result tuples */
    int         plan_width;     /* estimated average width of result tuples */
    
    List       *targetlist;     /* target list to be computed */
    List       *qual;           /* implicitly-ANDed qual conditions */
    struct Plan *lefttree;      /* input plan nodes */
    struct Plan *righttree;
    
    /* other fields... */
} Plan;
```

### Main Entry Points

The planner's main entry points:

```c
/* Main planner entry point */
PlannedStmt *planner(Query *parse, const char *query_string, int cursorOptions,
                  ParamListInfo boundParams);

/* Create all possible access paths */
void set_base_rel_paths(PlannerInfo *root, RelOptInfo *rel);

/* Create all possible join paths */
void add_paths_to_joinrel(PlannerInfo *root, RelOptInfo *joinrel,
                       RelOptInfo *outerrel, RelOptInfo *innerrel,
                       JoinType jointype, SpecialJoinInfo *sjinfo,
                       List *restrictlist);
```

## Common Modification Patterns

### Adding a New Join Method

1. Define a new path type in `include/nodes/pathnodes.h`:
   ```c
   typedef enum PathType
   {
       /* existing types... */
       T_MyJoinPath
   } PathType;
   ```

2. Create a structure for your path in `include/nodes/pathnodes.h`:
   ```c
   typedef struct MyJoinPath
   {
       JoinPath    jpath;      /* inherit from JoinPath */
       /* add your specific fields */
   } MyJoinPath;
   ```

3. Implement a path creation function in `optimizer/path/joinpath.c`:
   ```c
   MyJoinPath *
   create_myjoin_path(PlannerInfo *root, RelOptInfo *joinrel,
                     RelOptInfo *outerrel, RelOptInfo *innerrel,
                     JoinType jointype, SpecialJoinInfo *sjinfo,
                     List *restrictlist)
   {
       MyJoinPath *pathnode = makeNode(MyJoinPath);
       
       /* Initialize generic join fields */
       /* Initialize cost estimates */
       /* Initialize myjoin-specific fields */
       
       return pathnode;
   }
   ```

4. Add path cost estimation in `optimizer/path/costsize.c`:
   ```c
   void
   cost_myjoin(MyJoinPath *path, PlannerInfo *root,
              RelOptInfo *joinrel, RelOptInfo *outerrel,
              RelOptInfo *innerrel, JoinType jointype,
              List *restrictlist)
   {
       /* Implement cost estimation logic */
       /* Set path->jpath.path.startup_cost and path->jpath.path.total_cost */
   }
   ```

5. Create a plan conversion function in `optimizer/plan/createplan.c`:
   ```c
   Plan *
   create_myjoin_plan(PlannerInfo *root, MyJoinPath *best_path)
   {
       MyJoin *plan = makeNode(MyJoin);
       
       /* Set generic plan fields */
       /* Set join-specific plan fields */
       /* Set myjoin-specific fields */
       
       return (Plan *) plan;
   }
   ```

6. Call your path creation function in `add_paths_to_joinrel` in `joinpath.c`.

### Adding a New Scan Method

1. Define a path type and structure in `include/nodes/pathnodes.h`

2. Implement a path creation function in `optimizer/path/allpaths.c`:
   ```c
   void
   create_myscan_path(PlannerInfo *root, RelOptInfo *rel)
   {
       MyScanPath *pathnode = makeNode(MyScanPath);
       
       /* Initialize path fields */
       /* Compute costs */
       
       add_path(rel, (Path *) pathnode);
   }
   ```

3. Call your path creation function in `set_base_rel_paths` in `allpaths.c`

### Improving Cost Estimation

1. Identify the relevant cost model in `optimizer/path/costsize.c`

2. Modify or extend the cost formulas:
   ```c
   void
   cost_seqscan(Path *path, PlannerInfo *root, RelOptInfo *baserel,
               ParamPathInfo *param_info)
   {
       /* Original formula or your improved version */
       cost = ...
       
       path->startup_cost = ...
       path->total_cost = ...
   }
   ```

3. Consider adding new statistics or parameters to improve cost accuracy

### Adding a New Optimization Rule

1. Identify the appropriate phase for your optimization rule

2. For a join reordering rule, modify `optimizer/path/joinrels.c`

3. For a scan selection rule, modify `optimizer/path/allpaths.c`

4. For a query transformation rule, modify `optimizer/prep/`

## Important Considerations

### Paths vs Plans

- **Paths** are planning objects used only during optimization
- **Plans** are execution objects passed to the executor
- Each path type typically has a corresponding plan type

### Cost Model

The cost model considers:
- **I/O costs**: Reading/writing data from disk
- **CPU costs**: Processing tuples in memory
- **Startup cost**: Cost before first tuple is produced
- **Total cost**: Cost to produce all tuples

Parameters controlling the cost model are in `include/optimizer/cost.h` and can be tuned via GUC parameters.

### Statistics and Selectivity

The optimizer relies on statistics gathered by ANALYZE:
- Column value histograms
- Most common values
- Table sizes
- Index cardinality

Selectivity estimation functions in `clausesel.c` use these statistics to estimate how many rows will match predicates.

### Join Ordering Algorithm

PostgreSQL uses a dynamic programming algorithm for join ordering when the number of tables is less than `geqo_threshold`. For queries with more tables, it uses the Genetic Query Optimizer (GEQO).

### Interesting Orders

PostgreSQL tracks "interesting orders" (sort orderings that might be useful) through PathKeys. These help determine when a sort can be avoided because a path already produces data in the needed order.

## Building and Testing

To rebuild after modifying the optimizer:

```bash
cd src/backend/optimizer
make clean
make
```

To test your changes:

1. Rebuild PostgreSQL
   ```bash
   make
   make install
   ```

2. Use EXPLAIN to see how your changes affect query plans
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) SELECT ...
   ```

3. Create regression tests in `src/test/regress/`

4. Test your changes against the TPC-H or TPC-DS benchmarks

## Debugging Tips

1. Use `debug_print_plan` to print plans to server log
2. Use `debug_pretty_print` to make the output more readable
3. Set `client_min_messages` to DEBUG to see more info
4. Add your own `elog(DEBUG, ...)` statements
5. Use GDB to step through optimizer code

## Common Pitfalls

- **Correctness**: Ensure optimization transformations preserve query semantics
- **Cost Model**: Changes to cost formulas can have wide-ranging effects
- **Performance**: Optimizer code runs frequently, so keep it efficient
- **Memory Management**: Use appropriate memory contexts
- **Portability**: Consider different hardware and operating systems
- **Backward Compatibility**: Be cautious when changing optimizer behavior

## Additional Resources

- PostgreSQL Internals Documentation: https://www.postgresql.org/docs/current/planner-optimizer.html
- Source Code Comments: The `README` file in `src/backend/optimizer/`
- PostgreSQL Wiki: https://wiki.postgresql.org/wiki/Query_Planning
- Performance Tuning Documentation: https://www.postgresql.org/docs/current/performance-tips.html
- The PostgreSQL mailing lists, especially pgsql-hackers
