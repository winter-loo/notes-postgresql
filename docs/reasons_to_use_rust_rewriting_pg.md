1. null character being not allowed in utf-8 string is a relic. See: https://www.commandprompt.com/blog/null-characters-workarounds-arent-good-enough/

2. Add a new type requires adding all functions which could support this type, say `sum(point)` aggregate function.

3. Should use an ordered_map for the following hash join:

```sql
create table t1(a int, b int);
create table t2(a int, b int);
insert into t1 select i, i from generate_series(1, 1000) g(i);
insert into t2 select i, i from generate_series(1, 1000000) g(i);
-- join and order have the same column
explain (costs off) select * from t1 join t2 on t1.a = t2.a order by t1.a;
```

Consider using btree(like in Rust) to implement the ordered map instead of currently using hash map.