A canonical representation of an expression in PostgreSQL refers to a standardized, normalized form that ensures semantically equivalent expressions have the same structure regardless of how they were originally written. Let me explain this concept in more detail.

# What is a Canonical Representation?

A canonical representation is a unique, standardized form for an expression that follows specific rules to ensure consistency. When expressions are converted to their canonical form:

1. Equivalent expressions look identical - Different ways of writing the same logical expression will be transformed to have the same structure
2. Consistent ordering - Terms in commutative operations are arranged in a consistent order
3. Standardized patterns - Common patterns are transformed into standard forms

# How PostgreSQL Creates Canonical Representations

Looking at the PostgreSQL code, several mechanisms work together to create canonical representations:

1. AND/OR Flattening

The code you're currently viewing in prepqual.c shows part of this process. PostgreSQL flattens nested AND/OR expressions into N-argument form:

```c
/* Flatten any ANDs introduced just below here */
andlist = pull_ands(andlist);
```

This ensures that instead of having a tree like:

```txt
AND
 ├── A
 └── AND
      ├── B
      └── C
```

You get a flattened form:

```txt
AND
 ├── A
 ├── B
 └── C
```

2. Term Reordering

PostgreSQL reorders terms in commutative operations (AND, OR, +, *, etc.) according to consistent rules. For example, constants might be moved to the right side of comparisons, or terms might be sorted by their internal representation.


3. Expression Simplification

The optimizer applies various simplification rules:

* x AND TRUE → x
* x OR FALSE → x
* x AND FALSE → FALSE
* x OR TRUE → TRUE
* NOT (NOT x) → x

4. Predicate Normalization

Comparison predicates are standardized. For example:

`5 > column` might be rewritten as `column < 5`
`column != 5` might be rewritten as `NOT (column = 5)`

# Example of Canonicalization

For the expression in your example:

```sql
EXTRACT(YEAR FROM order_date) = 2023 AND EXTRACT(YEAR FROM order_date) > EXTRACT(YEAR FROM shipped_date)
```

The canonicalization process would:

1. Identify the repeated subexpression EXTRACT(YEAR FROM order_date)
2. Ensure both instances have identical internal representation (same operator precedence, same argument ordering)
3. Flatten any nested AND/OR structures
4. Apply any applicable simplification rules
5. Ensure consistent ordering of terms

The resulting canonical form makes it easier for the optimizer to recognize that EXTRACT(YEAR FROM order_date) appears twice and can be computed once.