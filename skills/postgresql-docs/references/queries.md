# Queries: NULL logic, joins, grouping, windows, CTEs, paging

## Contents
- NULL logic
- Conditional expressions and subqueries
- Joins
- Grouping and aggregates
- Window functions
- ORDER BY, LIMIT, set operations
- WITH queries (CTEs)

## NULL logic

- `x NOT IN (list or subquery)` is NULL, never true, as soon as the list holds a NULL and nothing matches. Express the condition positively, or exclude the NULLs. https://www.postgresql.org/docs/18/functions-comparisons.html https://www.postgresql.org/docs/18/functions-subquery.html
- `x = ANY (array)` is NULL (not false) when nothing matches and an element is NULL. An empty array gives false for ANY, true for ALL. https://www.postgresql.org/docs/18/functions-comparisons.html
- Null-safe comparison: `a IS [NOT] DISTINCT FROM b`, which never returns NULL. https://www.postgresql.org/docs/18/functions-comparisons.html
- Row comparison `ROW(a,b) < ROW(c,d)` compares left to right and stops at the first unequal or null pair (a null in that pair makes the result NULL). https://www.postgresql.org/docs/18/functions-comparisons.html
- `GREATEST` and `LEAST` ignore NULL arguments, unlike the SQL standard and some other databases. https://www.postgresql.org/docs/18/functions-conditional.html
- `concat()` and `concat_ws()` ignore NULL arguments. https://www.postgresql.org/docs/18/functions-string.html

## Conditional expressions and subqueries

- CASE evaluates only the branch it needs, but a constant like `1/0` can still fail at planning time inside an unreached branch. https://www.postgresql.org/docs/18/functions-conditional.html
- Aggregates are computed before CASE in SELECT and HAVING, so CASE cannot skip an aggregate. https://www.postgresql.org/docs/18/sql-select.html
- `NULLIF(1, 2.2)` returns numeric: the result takes the type of the implied `=` operator. https://www.postgresql.org/docs/18/functions-conditional.html
- Avoid side effects (sequence calls) in an EXISTS subquery: it runs only long enough to know whether a row exists, so whether the side effects occur is unpredictable. https://www.postgresql.org/docs/18/functions-subquery.html

## Joins

- In an outer join a condition in ON filters before joining and keeps unmatched rows, the same condition in WHERE removes them. https://www.postgresql.org/docs/18/queries-table-expressions.html
  ```sql
  SELECT * FROM t1 LEFT JOIN t2 ON t1.num = t2.num AND t2.value = 'xxx';
  ```
- Avoid `NATURAL JOIN`: a column added later with the same name silently joins too. `USING` is safe against that. https://www.postgresql.org/docs/18/queries-table-expressions.html
- `LATERAL` lets a FROM item reference earlier items. Keep rows with no lateral result using `LEFT JOIN LATERAL ... ON true`. https://www.postgresql.org/docs/18/queries-table-expressions.html
  ```sql
  SELECT m.name
  FROM manufacturers m LEFT JOIN LATERAL get_product_names(m.id) pname ON true
  WHERE pname IS NULL;
  ```
- An alias hides the table name: `FROM my_table AS m WHERE my_table.a > 5` is wrong. https://www.postgresql.org/docs/18/queries-table-expressions.html
- `WITH ORDINALITY` numbers the rows of a set-returning function. https://www.postgresql.org/docs/18/queries-table-expressions.html

## Grouping and aggregates

- Ungrouped columns are allowed in SELECT only when the table's PRIMARY KEY is in GROUP BY (functional dependency). https://www.postgresql.org/docs/18/sql-select.html
- Every aggregate except `count` returns NULL over zero rows: `sum` gives NULL, not 0, and `array_agg` gives NULL, not `{}`. Wrap in `COALESCE`. https://www.postgresql.org/docs/18/functions-aggregate.html
- Input order of `array_agg`, `string_agg`, `jsonb_agg` is unspecified. Write `array_agg(x ORDER BY y)`. A sorted subquery can be reordered by later joins. https://www.postgresql.org/docs/18/functions-aggregate.html
- `count(*)` on a whole table costs time proportional to its size. https://www.postgresql.org/docs/18/functions-aggregate.html
- `sum(integer)` returns bigint, `sum(bigint)` and `avg(integer)` return numeric. https://www.postgresql.org/docs/18/functions-aggregate.html
- In `ROLLUP`, `CUBE` and `GROUPING SETS` output, a rolled-up column is NULL like a real NULL. Tell them apart with `GROUPING()`. https://www.postgresql.org/docs/18/queries-table-expressions.html https://www.postgresql.org/docs/18/functions-aggregate.html
- `DISTINCT ON (a)` keeps the first row of each group: its ORDER BY must start with `a`. https://www.postgresql.org/docs/18/sql-select.html
  ```sql
  SELECT DISTINCT ON (location) location, time, report
      FROM weather_reports
      ORDER BY location, time DESC;
  ```

## Window functions

- The default frame ends at the last peer of the current row, so `last_value()` usually returns the current row. Write `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`. https://www.postgresql.org/docs/18/functions-window.html
- `sum(x) OVER (ORDER BY y)` is a running total. Without ORDER BY it covers the whole partition. https://www.postgresql.org/docs/18/functions-window.html
- `ROWS` frames are unpredictable unless ORDER BY is unique. `rank` leaves gaps, `dense_rank` does not, `row_number` is always unique. https://www.postgresql.org/docs/18/sql-select.html https://www.postgresql.org/docs/18/functions-window.html
- `IGNORE NULLS` and `FROM LAST` are not implemented. https://www.postgresql.org/docs/18/functions-window.html
- Window output order is not guaranteed: add a top-level ORDER BY. https://www.postgresql.org/docs/18/queries-table-expressions.html

## ORDER BY, LIMIT, set operations

- Without ORDER BY the order is unspecified, even between two runs of the same LIMIT query. Paging needs an ORDER BY on a unique key. https://www.postgresql.org/docs/18/queries-limit.html
- Large OFFSET values are slow: skipped rows are still computed. https://www.postgresql.org/docs/18/queries-limit.html
- NULLs sort last in ASC and first in DESC. Use `NULLS FIRST` or `NULLS LAST` to override. https://www.postgresql.org/docs/18/queries-order.html
- `FETCH FIRST n ROWS WITH TIES` returns the ties of the last row. It needs ORDER BY. https://www.postgresql.org/docs/18/sql-select.html
- UNION, INTERSECT and EXCEPT remove duplicate rows unless ALL is given (`UNION ALL`). https://www.postgresql.org/docs/18/queries-union.html
- A trailing LIMIT or ORDER BY applies to the whole UNION. Parenthesize one branch to limit it. https://www.postgresql.org/docs/18/queries-union.html
- Write `AS` before every output alias: it may be omitted only when the name is not a PostgreSQL keyword, and writing it always protects against future keywords. https://www.postgresql.org/docs/18/sql-select.html

## WITH queries (CTEs)

- A side-effect-free CTE referenced once is inlined. Referenced twice, it is computed once. `AS MATERIALIZED` forces a separate computation (optimization fence), `AS NOT MATERIALIZED` forces inlining. https://www.postgresql.org/docs/18/queries-with.html
  ```sql
  WITH w AS NOT MATERIALIZED (SELECT * FROM big_table)
  SELECT * FROM w AS w1 JOIN w AS w2 ON w1.key = w2.ref
  WHERE w2.key = 123;
  ```
- Data-modifying CTEs run exactly once to completion, all on the same snapshot: they cannot see each other's changes. Pass rows only through RETURNING. Never modify the same row twice in one statement. https://www.postgresql.org/docs/18/queries-with.html
  ```sql
  WITH moved_rows AS (
      DELETE FROM products
      WHERE "date" >= '2010-10-01' AND "date" < '2010-11-01'
      RETURNING *
  )
  INSERT INTO products_log SELECT * FROM moved_rows;
  ```
- A locking clause on the main query does not lock rows read by its CTEs. Put `FOR UPDATE` inside the CTE. https://www.postgresql.org/docs/18/sql-select.html
- Recursive CTEs: guard against cycles with the `CYCLE` clause and order with `SEARCH DEPTH FIRST BY`. The default output order is an implementation detail. https://www.postgresql.org/docs/18/queries-with.html
  ```sql
  WITH RECURSIVE search_graph(id, link, data, depth) AS (
      SELECT g.id, g.link, g.data, 1 FROM graph g
    UNION ALL
      SELECT g.id, g.link, g.data, sg.depth + 1
      FROM graph g, search_graph sg WHERE g.id = sg.link
  ) CYCLE id SET is_cycle USING path
  SELECT * FROM search_graph;
  ```
