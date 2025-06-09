# Common Subexpression Elimination in PostgreSQL

## Practical Example

Consider a query like:
```sql
SELECT * FROM orders 
WHERE 
  EXTRACT(YEAR FROM order_date) = 2023 AND 
  EXTRACT(YEAR FROM order_date) > EXTRACT(YEAR FROM shipped_date);
```

Without Common Subexpression Elimination (CSE), PostgreSQL would compute `EXTRACT(YEAR FROM order_date)` twice. With CSE:

1. The optimizer identifies that `EXTRACT(YEAR FROM order_date)` appears twice
2. It creates a computational plan that calculates this expression only once
3. The result is reused in both the first equality comparison and the second inequality comparison

## Implementation Details

PostgreSQL implements CSE primarily in the expression preprocessing phase during query planning. The implementation involves several key components:

### 1. Expression Tree Canonicalization

Before identifying common subexpressions, PostgreSQL normalizes expression trees into [a canonical form](./canonical_form_of_expressions.md). This makes it easier to detect when two expressions are semantically equivalent, even if they were written differently in the original query.

### 2. Expression Hashing and Matching

The optimizer uses a hash-based approach to identify duplicate expressions. When preprocessing expressions, it maintains a hash table of previously seen expressions. For each new expression:
- It computes a hash value based on the expression structure
- Checks if an equivalent expression already exists in the hash table
- If found, it replaces the current expression with a reference to the previously computed result

### 3. Implementation in ExecInitExpr

The executor's expression initialization routines (in `execExpr.c`) detect when the same subexpression appears multiple times in a query execution plan. Instead of creating separate execution structures for each occurrence, it creates a single execution structure and shares it across all occurrences.


### 5. Optimization Scope

CSE is applied at different levels:
- Within a single query level (most common)
- Across subqueries when possible
- Within complex expressions in WHERE, HAVING, and SELECT clauses

### 6. Integration with Other Optimizations

CSE works in conjunction with other optimizations:
- Constant folding (pre-evaluating constant expressions)
- Expression simplification (algebraic simplifications)
- Predicate pushdown (moving filtering operations earlier in the plan)

## Performance Impact

The performance benefits of CSE are most noticeable in:
- Queries with complex expressions that appear multiple times
- Queries involving expensive function calls
- Analytical queries with multiple references to the same calculated values

By eliminating redundant calculations, CSE reduces:
- CPU usage for expression evaluation
- Memory overhead for storing intermediate results
- Overall query execution time, especially for complex expressions

This optimization is particularly valuable in data warehousing and analytical workloads where complex expressions are common and often repeated across different parts of the same query.