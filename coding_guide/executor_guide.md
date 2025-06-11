# PostgreSQL Executor Modification Guide

## Overview

The PostgreSQL executor is responsible for running the query plans created by the optimizer. It implements a volcano/iterator-style execution model where each node pulls data from its children on demand. This guide provides essential information for developers who want to understand and modify the PostgreSQL executor.

## Architecture

The PostgreSQL executor follows a hierarchical architecture with several key components:

- **Executor State**: Tracks execution state and provides context for nodes
- **Plan State Nodes**: Represent the runtime state of plan nodes
- **Tuple Slots**: Hold tuples as they move through the execution tree
- **Expression Evaluation**: Evaluates expressions, predicates, and projections
- **Result Processing**: Delivers results to the client or calling process

The execution process follows these steps:
1. Initialize executor state from the plan tree
2. Start up each node in the execution tree (top to bottom)
3. Execute by repeatedly pulling tuples up the tree (bottom to top)
4. Clean up resources when execution completes or is canceled

## Key Files and Their Purposes

The executor code is primarily located in the `executor` directory:

| Directory | Purpose |
|-----------|---------|
| `executor` | Core executor functionality |
| `executor/nodeXXX.c` | Implementation of specific executor nodes |
| `executor/instrument.c` | Execution statistics collection |

Key files include:

| File | Location | Description |
|------|----------|-------------|
| `execMain.c` | `src/backend/executor` | Main entry point for executor |
| `execProcnode.c` | `src/backend/executor` | Central dispatch for different node types |
| `execQual.c` | `src/backend/executor` | Qualification (filter) evaluation |
| `execTuples.c` | `src/backend/executor` | Tuple handling |
| `execExpr.c` | `src/backend/executor` | Expression evaluation |
| `nodeScan.c` | `src/backend/executor` | Base functionality for scan nodes |
| `nodeHash.c` | `src/backend/executor` | Hash table implementation |
| `nodeHashjoin.c` | `src/backend/executor` | Hash join implementation |
| `nodeNestloop.c` | `src/backend/executor` | Nested loop join implementation |
| `nodeMergejoin.c` | `src/backend/executor` | Merge join implementation |
| `nodeAgg.c` | `src/backend/executor` | Aggregation implementation |

## Core Concepts and Data Structures

### EState

`EState` (Executor State) tracks global information about query execution:

```c
typedef struct EState
{
    NodeTag     type;           /* identifies as an EState */
    
    /* query resource management */
    MemoryContext es_query_cxt; /* per-query memory context */
    
    /* tuple table */
    TupleTable  es_tupleTable;  /* array of TupleTableSlots */
    
    /* result relation info */
    List       *es_result_relations; /* list of ResultRelInfo */
    ResultRelInfo *es_result_relation_info; /* currently active result rel */
    
    /* subplan info - for CTE and initplans */
    List       *es_subplanstates; /* List of PlanState nodes for subplans */
    
    /* other important fields... */
    JunkFilter *es_junkFilter;  /* junk filter for result tuples */
    bool        es_usesFdwForReturning; /* foreign tables involved? */
    
    /* features used during execution */
    bool        es_instrument;  /* collect instrumentation info */
    bool        es_finished;    /* set true when query is done */
} EState;
```

### PlanState

`PlanState` is the base structure for all executor node states:

```c
typedef struct PlanState
{
    NodeTag     type;           /* identifies node type in executor */
    Plan       *plan;           /* associated Plan node */
    EState     *state;          /* back link to top-level execution state */
    
    /* node's startup/cleanup function callbacks */
    ExecProcNodeMtd ExecProcNode; /* function pointer to node's main function */
    ExecProcNodeMtd ExecProcNodeReal; /* actual function, if above is a wrapper */
    
    Instrumentation *instrument;/* runtime statistics, if requested */
    
    /* tuple processing state */
    TupleTableSlot *ps_ResultTupleSlot; /* slot for node's result tuples */
    ExprContext *ps_ExprContext; /* expression context for node */
    ProjectionInfo *ps_ProjInfo; /* info for performing projections */
    
    /* initialization/cleanup info */
    bool        chgParam;       /* parameters have changed? */
    struct PlanState *lefttree; /* input plan node/nodes */
    struct PlanState *righttree;
    List       *initPlan;       /* Init SubPlanState nodes */
    List       *subPlan;        /* SubPlanState nodes */
    
    /* other important fields... */
} PlanState;
```

### TupleTableSlot

`TupleTableSlot` holds a tuple during execution:

```c
typedef struct TupleTableSlot
{
    NodeTag     type;           /* always T_TupleTableSlot */
    bool        tts_isempty;    /* true if slot contains no valid tuple */
    bool        tts_shouldFree; /* should pfree tts_tuple? */
    bool        tts_shouldFreeMem; /* should pfree tts_values/nulls arrays? */
    
    /* The tuple itself */
    HeapTuple   tts_tuple;      /* physical tuple (may be NULL) */
    
    /* Deconstructed tuple attributes */
    TupleDesc   tts_tupleDescriptor; /* descriptor for the tuple */
    Datum      *tts_values;     /* attribute values */
    bool       *tts_isnull;     /* null flags */
    
    /* Table access information */
    TableScanDesc tts_tableOid; /* table scan to fetch from */
    bool        tts_nvalid;     /* # of valid values/nulls */
    
    /* other important fields... */
} TupleTableSlot;
```

> [!note] TupleTableSlot
> 
> ```
> ┌─────────────────────────────────┐                ┌────────────────────────────────┐
> │          TupleTableSlot         │                │       TupleTableSlotOps        │
> ├─────────────────────────────────┤                ├────────────────────────────────┤
> │ NodeTag type                    │                │ size_t base_slot_size          │
> │ uint16 tts_flags                │◄───refers to───┤ void (*init)()                 │
> │ AttrNumber tts_nvalid           │                │ void (*release)()              │
> │ const TupleTableSlotOps *tts_ops│                │ void (*clear)()                │
> │ TupleDesc tts_tupleDescriptor   │                │ void (*getsomeattrs)()         │
> │ Datum *tts_values               │                │ void (*materialize)()          │
> │ bool *tts_isnull                │                │ void (*copyslot)()             │
> │ MemoryContext tts_mcxt          │                │ HeapTuple (*get_heap_tuple)()  │
> │ ItemPointerData tts_tid         │                │ MinimalTuple (*get_minimal)()  │
> │ Oid tts_tableOid                │                │ void (*copy_heap_tuple)()      │
> └─────────────────────────────────┘                │ bool (*current_xact_tuple)()   │
>                  ▲                                 └────────────────────────────────┘
>                  │                                           ▲
>      ┌───────────┼─────────────────┐                         │
>      │           │                 │                         │
> ┌────┴──────┐ ┌──┴───────┐   ┌─────┴────────┐      ┌─────────┴────────────┐
> │           │ │          │   │              │      │                      │
> │VirtualTTS │ │ HeapTTS  │   │MinimalTTS    │      │ Implementation       │
> │           │ │          │   │              │      │ Instances:           │
> └───────────┘ └──────────┘   └──────────────┘      │                      │
>                    ▲                               │ TTSOpsVirtual        │
>                    │                               │ TTSOpsHeapTuple      │
>          ┌─────────┴───────┐                       │ TTSOpsMinimalTuple   │
>          │                 │                       │ TTSOpsBufferHeapTuple│
> ┌────────┴────────────┐    │                       └──────────────────────┘
> │                     │    │
> │ BufferHeapTTS       │    │          ┌────────────────────┐      ┌─────────────┐
> │                     │    └─uses─────┤     HeapTuple      │◄─────┤  TupleDesc  │
> └─────────────────────┘               └────────────────────┘      └─────────────┘
>                                                 │                        ▲
>                                                 │                        │
>                                                 ▼                        │
>                               ┌────────────────────────┐                 │
>                               │    MinimalTuple        │─────────────────┘
>                               └────────────────────────┘
> ```

This diagram illustrates the object-oriented relationships around `TupleTableSlot` in PostgreSQL:

1. **TupleTableSlot** - The base structure containing metadata about a tuple and referencing a specific operations implementation

2. **TupleTableSlotOps** - Interface defining operations that can be performed on tuple slots with various implementations

3. **Concrete Slot Implementations**:
   - **VirtualTupleTableSlot**: For "virtual" tuples(not own data but reference the underlying tuple data)
   - **HeapTupleTableSlot**: For regular heap tuples in memory
   - **BufferHeapTupleTableSlot**: For tuples in disk buffer pages
   - **MinimalTupleTableSlot**: For minimal tuples (compact representation)

4. **Operation Implementations**:
   - TTSOpsVirtual, TTSOpsHeapTuple, TTSOpsMinimalTuple, TTSOpsBufferHeapTuple

5. **Related Structures**:
   - HeapTuple, MinimalTuple, TupleDesc

### ExprContext

`ExprContext` provides a context for expression evaluation:

```c
typedef struct ExprContext
{
    NodeTag     type;           /* identifies node type */
    
    /* Memory management */
    TupleTableSlot *ecxt_scantuple;  /* current input tuple for evaluation */
    TupleTableSlot *ecxt_innertuple; /* inner tuple for joins */
    TupleTableSlot *ecxt_outertuple; /* outer tuple for joins */
    
    /* Per-tuple memory context for expression evaluation */
    MemoryContext ecxt_per_tuple_memory; /* reset for each tuple */
    
    /* other important fields... */
} ExprContext;
```

### Main Entry Points

The executor's main entry points:

```c
/* Main executor entry point */
TupleTableSlot *
ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count,
          bool execute_once);

/* Initialize the executor */
void
ExecutorStart(QueryDesc *queryDesc, int eflags);

/* Clean up after execution */
void
ExecutorFinish(QueryDesc *queryDesc);

/* Release executor resources */
void
ExecutorEnd(QueryDesc *queryDesc);

/* Execute a single node */
TupleTableSlot *
ExecProcNode(PlanState *node);
```

## Common Modification Patterns

### Adding a New Executor Node

1. Define a new plan node type in `include/nodes/nodes.h`:
   ```c
   typedef enum NodeTag
   {
       /* existing types... */
       T_MyCustomPlan,
   } NodeTag;
   ```

2. Define a plan node structure in `include/nodes/plannodes.h`:
   ```c
   typedef struct MyCustomPlan
   {
       Plan        plan;        /* inherit from Plan */
       /* add your specific fields */
   } MyCustomPlan;
   ```

3. Define a plan state structure in `include/nodes/execnodes.h`:
   ```c
   typedef struct MyCustomPlanState
   {
       PlanState   ps;          /* inherit from PlanState */
       /* add your specific runtime state fields */
   } MyCustomPlanState;
   ```

4. Create a new file `nodeMyCustom.c` in `src/backend/executor/` with these functions:
   ```c
   /* Initialize the executor node */
   MyCustomPlanState *
   ExecInitMyCustom(MyCustomPlan *node, EState *estate, int eflags)
   {
       MyCustomPlanState *state = makeNode(MyCustomPlanState);
       
       /* Initialize base PlanState fields */
       state->ps.plan = (Plan *)node;
       state->ps.state = estate;
       state->ps.ExecProcNode = ExecMyCustom;
       
       /* Initialize child nodes */
       state->ps.lefttree = ExecInitNode(outerPlan(node), estate, eflags);
       
       /* Allocate tuple slot for this node */
       ExecInitResultTupleSlot(estate, &state->ps);
       
       /* Initialize node-specific fields */
       /* ... */
       
       return state;
   }

   /* Main execution function for the node */
   TupleTableSlot *
   ExecMyCustom(PlanState *pstate)
   {
       MyCustomPlanState *node = castNode(MyCustomPlanState, pstate);
       TupleTableSlot *resultSlot = node->ps.ps_ResultTupleSlot;
       
       /* Implement node-specific execution logic */
       /* ... */
       
       /* Return result tuple */
       return resultSlot;
   }

   /* End execution and free resources */
   void
   ExecEndMyCustom(MyCustomPlanState *node)
   {
       /* Free node-specific resources */
       
       /* Clean up child nodes */
       if (node->ps.lefttree)
           ExecEndNode(node->ps.lefttree);
   }
   ```

5. Add function declarations to `include/executor/nodeMyCustom.h`:
   ```c
   extern MyCustomPlanState *ExecInitMyCustom(MyCustomPlan *node, EState *estate, int eflags);
   extern TupleTableSlot *ExecMyCustom(PlanState *pstate);
   extern void ExecEndMyCustom(MyCustomPlanState *node);
   ```

6. Register the node in `execProcnode.c`:
   ```c
   /* In ExecInitNode() */
   case T_MyCustomPlan:
       result = (PlanState *) ExecInitMyCustom((MyCustomPlan *) node,
                                            estate, eflags);
       break;
   
   /* In ExecEndNode() */
   case T_MyCustomPlanState:
       ExecEndMyCustom((MyCustomPlanState *) node);
       break;
   ```

### Modifying Tuple Processing

When modifying how tuples are processed:

1. Understand the `ExecQual()` function in `execQual.c` for filter condition evaluation:
   ```c
   bool
   ExecQual(List *qual, ExprContext *econtext, bool resultForNullFlag)
   {
       /* Evaluate each expression in the qualification list */
       ListCell *l;
       
       /* If the qualification is empty, it's considered true */
       if (qual == NIL)
           return true;
           
       foreach(l, qual)
       {
           ExprState *clause = (ExprState *) lfirst(l);
           Datum result;
           bool isNull;
           
           /* Evaluate the expression */
           result = ExecEvalExpr(clause, econtext, &isNull, NULL);
           
           /* Return false if expression is false or null */
           if (isNull)
               return resultForNullFlag;
           else if (!DatumGetBool(result))
               return false;
       }
       
       /* All expressions evaluated to true, so the qualification passes */
       return true;
   }
   ```

2. For projection, use or modify `ExecProject()` in `execQual.c`:
   ```c
   TupleTableSlot *
   ExecProject(ProjectionInfo *projInfo)
   {
       /* Evaluate expressions and store results in the target slot */
       TupleTableSlot *slot = projInfo->pi_slot;
       ExprContext *econtext = projInfo->pi_exprContext;
       int numSimpleVars = projInfo->pi_numSimpleVars;
       
      /* Evaluate expressions */
      /* ... */
      
      /* Return the slot containing projected tuple */
      return slot;
   }
   ```

### Adding Runtime Metrics

1. Use the `Instrumentation` structure in `instrument.c`:
   ```c
   typedef struct Instrumentation
   {
       instr_time  starttime;   /* Start time of execution */
       instr_time  counter;     /* Accumulated execution time */
       double      firsttuple;  /* Time for first tuple of this cycle */
       double      tuplecount;  /* Tuples processed in this cycle */
       bool        running;     /* True if node currently running */
       bool        needs_timer; /* True if we need to time this node */
   } Instrumentation;
   ```

2. Track execution metrics in your executor node:
   ```c
   /* In your ExecMyCustom function */
   if (node->ps.instrument)
       InstrStartNode(node->ps.instrument);
       
   /* ... do work ... */
   
   if (node->ps.instrument)
       InstrStopNode(node->ps.instrument, 1 /* tuples processed */);
   ```

## Executor Execution Flow

The executor follows a pull-based (volcano/iterator) model:

1. `ExecutorRun()` is called by the client
2. It calls `ExecutePlan()` to iterate through the plan execution
3. `ExecutePlan()` repeatedly calls `ExecProcNode()` on the top node
4. Each node pulls tuples from its children as needed
5. Tuples flow up the tree, with each node applying its operations

A simple walk through nested loop join:

```c
/* Simplified nested loop join execution */
static TupleTableSlot *
ExecNestedLoop(PlanState *pstate)
{
    NestLoopState *nlstate = castNode(NestLoopState, pstate);
    PlanState *outerPlan = outerPlanState(nlstate);
    PlanState *innerPlan = innerPlanState(nlstate);
    
    /* Loop until we find a matching pair of tuples */
    for (;;)
    {
        /* Get the next outer tuple */
        if (nlstate->nl_NeedNewOuter)
        {
            /* Get new outer tuple, return NULL if no more tuples */
            TupleTableSlot *outerTupleSlot = ExecProcNode(outerPlan);
            if (TupIsNull(outerTupleSlot))
                return NULL; /* No more outer tuples */
                
            /* Reset inner scan for this outer tuple */
            ExecReScan(innerPlan);
            nlstate->nl_NeedNewOuter = false;
        }
        
        /* Get next inner tuple */
        TupleTableSlot *innerTupleSlot = ExecProcNode(innerPlan);
        if (TupIsNull(innerTupleSlot))
        {
            /* No more inner tuples, get new outer tuple */
            nlstate->nl_NeedNewOuter = true;
            continue;
        }
        
        /* Set up expression context for join qual evaluation */
        nlstate->js.ps.ps_ExprContext->ecxt_outertuple = outerTupleSlot;
        nlstate->js.ps.ps_ExprContext->ecxt_innertuple = innerTupleSlot;
        
        /* Evaluate join qualifications */
        if (ExecQual(nlstate->js.joinqual, nlstate->js.ps.ps_ExprContext, false))
        {
            /* Return projection result */
            return ExecProject(nlstate->js.ps.ps_ProjInfo);
        }
        
        /* Failed join qual, continue with next inner tuple */
    }
}
```

## Building and Testing

To rebuild after modifying the executor:

```bash
# From the PostgreSQL source directory
cd src/backend/executor
make
cd ../..
make install
```

For testing:

1. Write regression tests in `src/test/regress/` for functionality
2. Use the `EXPLAIN ANALYZE` command to verify execution behavior
3. Consider using the `pg_test_timing` utility for performance microbenchmarks

### Performance Analysis

To analyze executor performance:

1. Enable timing: `SET track_io_timing = on;`
2. Use `EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)` to see detailed execution metrics
3. For deeper analysis, use external tools:
   - `perf` on Linux: `perf record -g -p postgres_pid`
   - `dtrace` on macOS and Solaris

## Common Pitfalls

1. **Memory Context Management**: Always allocate node-specific data in the appropriate memory context to avoid leaks.
2. **TupleTableSlot Handling**: Be careful not to return a slot before you're done with it, as other nodes may modify it.
3. **ExprContext Reset**: For expression evaluation, reset the per-tuple memory context between tuples.
4. **Error Handling**: Use PG_TRY/PG_CATCH for operations that might throw errors to ensure proper cleanup.
5. **Resource Release**: Always pair resource acquisition with release, especially for executor nodes.

## Best Practices

1. Follow the volcano/iterator pattern consistently
2. Avoid materializing full result sets when streaming is possible
3. Consider parallelism implications for new executor nodes
4. Ensure proper cleanup in the ExecEnd function
5. Document performance characteristics and expected behavior
6. Keep nodes focused on a single responsibility
7. Add appropriate instrumentation for monitoring and debugging
