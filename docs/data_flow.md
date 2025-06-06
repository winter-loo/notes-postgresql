# PostgreSQL SQL Statement Processing: Data Flow Diagram

This document provides a visual representation of how data flows through PostgreSQL's key data structures during SQL statement processing. The diagram illustrates the transformation of a SQL statement from text to execution results.

## High-Level Data Flow

```mermaid
flowchart TD
    SQL[SQL String] --> Parser[Parser]
    Parser --> RawParse[Raw Parse Tree]
    RawParse --> Analyzer[Analyzer]
    Analyzer --> Query[Query Tree]
    Query --> Rewriter[Rule Rewriter]
    Rewriter --> RewrittenQuery[Rewritten Query Tree]
    RewrittenQuery --> Planner[Planner/Optimizer]
    Planner --> Plan[Plan Tree]
    Plan --> Executor[Executor]
    Executor --> Results[Result Tuples]
    Results --> Destination[Result Destination]
```

## Detailed Data Flow with Key Data Structures

```mermaid
flowchart TD
    %% External Interface
    Client[Client] -->|SQL text| Port[Port]
    Port -->|SQL text| StringInfo[StringInfo]
    
    %% Parsing Phase
    StringInfo -->|SQL text| Parser[Parser & Lexer]
    Parser -->|List of| RawStmt[RawStmt] 
    RawStmt -->|Parse tree nodes\ne.g., SelectStmt, InsertStmt| ParseState[ParseState]
    
    %% Analysis Phase
    ParseState -->|Resolves names,\ntypes, etc.| Analyzer[Analyzer]
    Analyzer -->|Produces| Query[Query]
    Query -->|Contains|\nRTEs[Range Table Entries\nRangeTblEntry]
    Query -->|Contains| TargetList[Target List\nTargetEntry]
    Query -->|Contains| JoinTree[Join Tree\nFromExpr]

    %% Rewriting Phase
    Query -->|Input to| Rewriter[Rule Rewriter]
    Rewriter -->|May produce\nmultiple| RewrittenQueries[Rewritten Queries]
    
    %% Planning Phase
    RewrittenQueries -->|Optimized by| PlannerGlobal[PlannerGlobal]
    PlannerGlobal -->|Contains| PlannerInfo[PlannerInfo]
    PlannerInfo -->|Analyzes|\nRelOptInfo[RelOptInfo]
    RelOptInfo -->|Generates|\nPaths[Path Options]
    Paths -->|Select best| Plan[Plan Tree]
    
    %% Plan Structure
    Plan -->|Tree of\noperators| PlanNodes[Plan Nodes\ne.g., SeqScan, HashJoin]
    
    %% Execution Phase
    Plan -->|Creates| EState[EState]
    EState -->|Contains| PlanState[PlanState Tree]
    PlanState -->|Contains| ExprContext[ExprContext]
    PlanState -->|Uses| TupleTableSlot[TupleTableSlot]
    TupleTableSlot -->|May hold| HeapTuple[HeapTuple/\nMinimalTuple]
    
    %% Data Access & Results
    PlanState -->|Accesses| Relation[Relation]
    Relation -->|Via| BufferDesc[Buffer Pool\nBufferDesc]
    PlanState -->|Produces| ResultTuples[Result Tuples]
    ResultTuples -->|Sent to| DestReceiver[DestReceiver]
    DestReceiver -->|E.g., Client,\nTable, Void| FinalDest[Final Destination]
    
    %% Memory Management
    TopMemoryContext[TopMemoryContext] -->|Parent of| TransactionContext[Transaction Context]
    TransactionContext -->|Parent of| QueryContext[Query Context\nes_query_cxt]
    QueryContext -->|Parent of| TempContexts[Temporary\nContexts]
    
    %% Transaction Management
    TransactionState[TransactionState] -->|Manages| Snapshot[Snapshot]
    Snapshot -->|Provides\nvisibility to| PlanState
    TransactionState -->|Manages| ResourceOwner[ResourceOwner]
    ResourceOwner -->|Tracks| Resources[Resources\ne.g., buffers, files]
```

## Key Structure Relationships during SQL Processing

### Parsing Stage
- **StringInfo** holds the SQL text
- **Parser** produces syntax tree with nodes like **SelectStmt**, **InsertStmt**, etc.
- These raw nodes are collected in a **List** structure within a **RawStmt**

### Analysis Stage
- **ParseState** is created to hold state during analysis
- Raw parse tree is transformed into a **Query** structure with:
  - **RangeTblEntry** list (relations referenced)
  - **TargetEntry** list (output columns)
  - **FromExpr** (join tree structure)
  - Various condition expressions

### Planning Stage
- **PlannerGlobal** and **PlannerInfo** hold state during planning
- **RelOptInfo** structures represent each base relation and join
- **Path** structures represent possible access methods
- The best path combination is transformed into a **Plan** tree

### Execution Stage
- **EState** holds the overall execution state
- Each plan node has a corresponding **PlanState** node with runtime state
- **TupleTableSlot** holds tuples as they flow between nodes
- **HeapTuple** or **MinimalTuple** represent actual row data
- **ExprContext** evaluates expressions on current tuples
- **DestReceiver** directs output to its final destination

### Memory Management
- All allocations happen in proper **MemoryContext** hierarchies
- The context hierarchy ensures memory is freed at appropriate times

## Data Flow Through Memory Management

```mermaid
flowchart TD
    TopMemoryContext -->|Persists for\nbackend lifetime| ErrorContext & MessageContext & CacheMemoryContext & TransactionContext
    TransactionContext -->|Persists for\ntransaction lifetime| QueryContext[Per-Query Context]
    QueryContext -->|Persists for\nquery lifetime| ExecContexts[Executor Contexts]
    ExecContexts -->|Created/deleted\nduring execution| PerNodeContexts[Per-Node Contexts]
    PerNodeContexts -->|Reset frequently| PerTupleContexts[Per-Tuple Contexts]
```

## Horizontal View of Query Processing Pipeline

```mermaid
flowchart LR
    SQL[SQL] -->|Parser| ParseTree[Parse Tree]
    ParseTree -->|Analyzer| Query[Query]
    Query -->|Rewriter| RewrittenQuery[Rewritten Query]
    RewrittenQuery -->|Planner| Plan[Plan]
    Plan -->|Executor| Results[Results]
```

Each stage in the pipeline transforms the representation of the query, from textual SQL to raw syntax trees, to analyzed query trees, to optimized execution plans, and finally to result tuples. The data structures detailed above facilitate these transformations and maintain state throughout the process.
