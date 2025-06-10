# PostgreSQL SQL Analyzer Modification Guide

## Overview

The SQL analyzer is a critical component of PostgreSQL's query processing pipeline, responsible for transforming raw parse trees (produced by the parser/lexer) into structured Query trees ready for planning and execution. This guide provides essential information for developers who want to understand and modify the PostgreSQL SQL analyzer.

## Architecture

The PostgreSQL SQL analyzer follows a modular architecture designed to handle various SQL statement types:

- **Core Analyzer (`analyze.c`)**: Entry point that dispatches different statement types to specialized handlers
- **Specialized Parsers**: Multiple modules that handle specific parts of SQL statements (expressions, clauses, etc.)
- **Parse States**: Context objects that track the state during analysis
- **Node Transformation**: Converting parse nodes into executable query nodes

The analysis process follows these steps:
1. Raw parse tree is received from the parser
2. The statement type is identified and dispatched to the appropriate transform function
3. Various parse_* helper modules process specific parts of the statement
4. A structured Query node is built with appropriate attributes
5. For DML statements, target relations are identified, and permissions are checked
6. For complex queries, subqueries and expressions are recursively analyzed
7. The final Query tree is produced, ready for planning and optimization

## Key Files and Their Purposes

| File | Location | Description |
|------|----------|-------------|
| `analyze.c` | `src/backend/parser/` | Main analyzer implementation and entry point |
| `analyze.h` | `src/include/parser/` | Public API for the analyzer |
| `parse_node.h` | `src/include/parser/` | Definition of ParseState and related structures |
| `parse_clause.c` | `src/backend/parser/` | Handlers for FROM, WHERE, GROUP BY clauses |
| `parse_expr.c` | `src/backend/parser/` | Expression analysis functions |
| `parse_target.c` | `src/backend/parser/` | Target list (SELECT columns) processing |
| `parse_relation.c` | `src/backend/parser/` | Table/view reference analysis |
| `parse_func.c` | `src/backend/parser/` | Function call analysis |
| `parse_agg.c` | `src/backend/parser/` | Aggregate function handling |
| `parse_coerce.c` | `src/backend/parser/` | Type coercion functions |
| `parsetree.h` | `src/include/parser/` | Query tree structure definition |

## Core Concepts and APIs

### Parse State

The ParseState structure maintains context during analysis:

```c
typedef struct ParseState
{
    /* Current parser state */
    struct ParseState *parentParseState;   /* stack link */
    const char *p_sourcetext;              /* source text of query */
    List       *p_rtable;                  /* range table */
    List       *p_joinexprs;               /* JOIN expressions */
    List       *p_joinlist;                /* join list (of RangeTblRef) */
    /* ... more fields ... */
} ParseState;
```

Key functions for managing parse state:
```c
ParseState *make_parsestate(ParseState *parentParseState);
void free_parsestate(ParseState *pstate);
```

### Entry Points

Main analysis entry points:

```c
/* Analyze a raw parse tree and transform it to Query form */
Query *parse_analyze_fixedparams(RawStmt *parseTree, 
                                const char *sourceText,
                                const Oid *paramTypes, 
                                int numParams, 
                                QueryEnvironment *queryEnv);

/* Analyze with variable parameters */
Query *parse_analyze_varparams(RawStmt *parseTree, 
                              const char *sourceText,
                              Oid **paramTypes, 
                              int *numParams,
                              QueryEnvironment *queryEnv);

/* Top-level statement transformation */
Query *transformTopLevelStmt(ParseState *pstate, RawStmt *parseTree);

/* Generic statement transformation */
Query *transformStmt(ParseState *pstate, Node *parseTree);
```

### Query Structure

The Query node is the main output of analysis:

```c
typedef struct Query
{
    NodeTag     type;           /* T_Query */
    CommandType commandType;    /* SELECT/INSERT/UPDATE/DELETE/etc */
    QuerySource querySource;    /* where did I come from? */
    uint32      queryId;        /* query identifier */
    bool        canSetTag;      /* do I set the command result tag? */
    /* ... many more fields for different query aspects ... */
} Query;
```

### Specialized Transformation Functions

Each statement type has a dedicated transform function:

```c
static Query *transformSelectStmt(ParseState *pstate, SelectStmt *stmt);
static Query *transformInsertStmt(ParseState *pstate, InsertStmt *stmt);
static Query *transformUpdateStmt(ParseState *pstate, UpdateStmt *stmt);
static Query *transformDeleteStmt(ParseState *pstate, DeleteStmt *stmt);
// And many more for other statement types
```

## Common Modification Patterns

### Adding Support for a New Statement Type

1. Add a new case in `transformStmt()` in `analyze.c`:
   ```c
   case T_MyNewStmt:
       result = transformMyNewStmt(pstate, (MyNewStmt *) parseTree);
       break;
   ```

2. Implement the transformation function:
   ```c
   static Query *
   transformMyNewStmt(ParseState *pstate, MyNewStmt *stmt)
   {
       Query *qry = makeNode(Query);
       qry->commandType = CMD_YOUR_COMMAND;
       
       /* Transform components */
       /* Set up range table */
       /* Process target lists */
       
       return qry;
   }
   ```

3. Update `stmt_requires_parse_analysis()` and `analyze_requires_snapshot()` if needed

### Adding New Expression Types

1. Update `transformExpr()` in `parse_expr.c` to handle your new node type:
   ```c
   case T_MyNewExpr:
       result = transformMyNewExpr(pstate, (MyNewExpr *) expr);
       break;
   ```

2. Implement the expression transformation function:
   ```c
   static Node *
   transformMyNewExpr(ParseState *pstate, MyNewExpr *expr)
   {
       /* Transform subexpressions */
       /* Apply type coercion if needed */
       /* Return transformed node */
   }
   ```

### Modifying SQL Clause Processing

1. Find the relevant parse_*.c file for the clause you want to modify
2. Update the corresponding transformation function:
   ```c
   /* For example, modifying GROUP BY in parse_clause.c */
   List *
   transformGroupClause(ParseState *pstate, List *grouplist,
                       List **targetlist, List *sortClause,
                       ParseExprKind exprKind)
   {
       /* Your modified logic here */
   }
   ```

### Adding a New Query Option

1. Modify the appropriate statement structure in `nodes/parsenodes.h`
2. Update the corresponding transform function to handle the new option
3. Modify the Query node structure if needed to store the option
4. Update the query execution path to respect the new option

## Important Considerations

### Type Coercion and Resolution

PostgreSQL relies heavily on type coercion during analysis. Key functions:

```c
/* In parse_coerce.c */
Node *coerce_type(ParseState *pstate, Node *node,
                 Oid inputTypeId, Oid targetTypeId,
                 int32 targetTypMod,
                 CoercionContext ccontext,
                 CoercionForm cformat,
                 int location);
```

When adding new expressions or modifying existing ones, ensure appropriate type handling.

### Range Table Management

The range table (rtable) is a list of `RangeTblEntry` nodes, each representing a table or subquery:

```c
/* Add a relation to the range table */
RangeTblEntry *addRangeTableEntry(ParseState *pstate,
                                 RangeVar *relation,
                                 Alias *alias,
                                 bool inh,
                                 bool inFromCl);
```

### Subquery Processing

When handling subqueries, use:

```c
Query *parse_sub_analyze(Node *parseTree, 
                       ParseState *parentParseState,
                       CommonTableExpr *parentCTE,
                       bool locked_from_parent,
                       bool resolve_unknowns);
```

### Security and Permissions

The analyzer performs preliminary permission checks during analysis. Be mindful of security implications when modifying code.

## Building and Testing

To rebuild the analyzer after making changes:

```bash
cd src/backend/parser
make clean
make
```

To test your changes:

1. Rebuild PostgreSQL
   ```bash
   make
   make install
   ```

2. Test with specific SQL statements that exercise your modifications
   ```bash
   psql -c "SELECT your_new_syntax;"
   ```

3. Run regression tests
   ```bash
   make check
   ```

4. Create new regression tests in `src/test/regress/` for your changes

## Common Pitfalls

- **Context Management**: Always properly initialize and free ParseState structures
- **Memory Management**: Use appropriate memory contexts to avoid leaks
- **Type Handling**: Ensure proper type coercion and compatibility checks
- **Error Reporting**: Use `ereport()` with appropriate error codes and positions
- **Name Resolution**: Be careful with relation and column name resolution precedence
- **Recursive Processing**: Handle recursive structures like CTEs properly
- **Performance Implications**: Analyzer changes can affect query planning and performance
- **Compatibility**: Ensure backward compatibility with existing SQL syntax

## Additional Resources

- PostgreSQL Parser/Analyzer Documentation: https://www.postgresql.org/docs/current/overview.html
- PostgreSQL Coding Style: https://www.postgresql.org/docs/current/source.html
- PostgreSQL Developer Wiki: https://wiki.postgresql.org/wiki/Developer_FAQ
- PostgreSQL Mailing Lists: https://www.postgresql.org/list/
- Source Code Documentation in `README` files within source directories
