# JSON and JSONB

## Contents
- Storing JSON
- Operators
- Indexing jsonb
- Modifying jsonb
- jsonpath
- SQL/JSON functions and JSON_TABLE
- Building and aggregating JSON

## Storing JSON

- Use `jsonb`. `json` keeps the exact text (whitespace, key order, duplicate keys), is faster to input but slower to process. `jsonb` supports indexing. https://www.postgresql.org/docs/18/datatype-json.html
- `jsonb` keeps only the last duplicate key, not key order. It rejects `\u0000` and numbers outside `numeric` range. https://www.postgresql.org/docs/18/datatype-json.html
- Any update locks the whole row: keep documents small and with a fairly fixed structure. https://www.postgresql.org/docs/18/datatype-json.html
- JSON `null` is not SQL NULL: `json_typeof('null')` returns the string `'null'`. https://www.postgresql.org/docs/18/functions-json.html

## Operators

- `->`, `->>`, `#>`, `#>>` return NULL, not an error, for a missing key or index. Array indexes start at 0 and negative ones count from the end. https://www.postgresql.org/docs/18/functions-json.html
- Comparison operators and `@>`, `?`, `||`, `-`, `#-`, `@?`, `@@` exist for `jsonb` only. https://www.postgresql.org/docs/18/functions-json.html
- `@>` is nested and structural: `'{"foo": {"bar": "baz"}}' @> '{"bar": "baz"}'` is false. Array containment ignores order and duplicates. https://www.postgresql.org/docs/18/datatype-json.html
- `?` checks only top-level keys or string array elements, not nested values. https://www.postgresql.org/docs/18/datatype-json.html
- Prefer one containment test on the whole column over extracting first: https://www.postgresql.org/docs/18/datatype-json.html
  ```sql
  SELECT doc->'site_name' FROM websites
    WHERE doc @> '{"tags":[{"term":"paris"}, {"term":"food"}]}';
  ```
- Subscripts return `jsonb`, so compare to JSON values: `WHERE col['key'] = '"value"'`. https://www.postgresql.org/docs/18/datatype-json.html

## Indexing jsonb

- Default: `CREATE INDEX ON api USING GIN (jdoc jsonb_path_ops);` It serves `@>`, `@?`, `@@` and is much smaller and faster than the default `jsonb_ops`. Use plain `USING GIN (jdoc)` only when you need key existence (`?`, `?|`, `?&`). https://www.postgresql.org/docs/18/datatype-json.html https://www.postgresql.org/docs/18/gin.html
- `jsonb_path_ops` stores no entry for empty structures like `{"a": {}}` (full index scan to find them) and does not support `.*` or `.**` in jsonpath. https://www.postgresql.org/docs/18/datatype-json.html
- An index is used only when the operator applies to the indexed expression itself. `jdoc -> 'tags' ? 'qui'` cannot use a GIN index on `jdoc`. Index the expression instead: https://www.postgresql.org/docs/18/datatype-json.html
  ```sql
  CREATE INDEX idxgintags ON api USING GIN ((jdoc -> 'tags'));
  ```
- B-tree and hash on jsonb only help whole-document equality. https://www.postgresql.org/docs/18/datatype-json.html

## Modifying jsonb

- `||` is not recursive: it merges top-level keys, right side wins. A non-array joined to an array is wrapped: `'{"a": "b"}' || '42'` gives `[{"a": "b"}, 42]`. https://www.postgresql.org/docs/18/functions-json.html
- `jsonb_set(target, path, value)` returns the target unchanged when an earlier path step is missing. Use `jsonb_set_lax` when the value may be NULL: its `null_value_treatment` is `'raise_exception'`, `'use_json_null'` (default), `'delete_key'` or `'return_target'`. https://www.postgresql.org/docs/18/functions-json.html
- `jsonb_insert` on an object key inserts only if the key is absent. https://www.postgresql.org/docs/18/functions-json.html
- Subscript assignment creates missing objects but errors if an intermediate value is a scalar. A NULL column becomes `{}` or `[]`. https://www.postgresql.org/docs/18/datatype-json.html
  ```sql
  UPDATE table_name SET jsonb_field['a']['b']['c'] = '1';
  ```
- `jsonb - 'key'` deletes a key, or every matching string in an array. `jsonb - 2` errors on non-arrays. https://www.postgresql.org/docs/18/functions-json.html
- `jsonb_strip_nulls(x, true)` also strips nulls inside arrays in PostgreSQL 18. https://www.postgresql.org/docs/18/functions-json.html

## jsonpath

- Pass values with the `vars` argument instead of building the path string: https://www.postgresql.org/docs/18/functions-json.html
  ```sql
  jsonb_path_exists(doc, '$.a[*] ? (@ >= $min && @ <= $max)', '{"min":2, "max":4}')
  ```
- `@?` takes a path and tests whether it returns anything. `@@` takes a predicate (`'$.a[*] > 2'`). Do not mix them. https://www.postgresql.org/docs/18/functions-json.html
- Lax mode (default) silently ignores missing keys and type mismatches and unwraps arrays. `strict` raises errors. Use `.**` only with `strict`: in lax mode it returns items twice. https://www.postgresql.org/docs/18/functions-json.html
- Date and time comparisons that need a zone work only in the `_tz` functions, which are STABLE and cannot be indexed. The plain functions are IMMUTABLE and error on such comparisons. https://www.postgresql.org/docs/18/functions-json.html
- `like_regex` patterns are JSON strings: double the backslashes (`"^\\d+$"`). https://www.postgresql.org/docs/18/functions-json.html

## SQL/JSON functions and JSON_TABLE

- `JSON_VALUE` is for one scalar: text by default, `RETURNING` for another type, SQL NULL for a JSON null. Use `JSON_QUERY` when an object or array can come back. https://www.postgresql.org/docs/18/functions-json.html
- Defaults: `NULL ON EMPTY` and `NULL ON ERROR` (errors are hidden). `JSON_EXISTS` defaults to `FALSE ON ERROR`. Write `ERROR ON ERROR` to see failures. https://www.postgresql.org/docs/18/functions-json.html
- `JSON_QUERY` with several results needs `WITH WRAPPER`. It keeps quotes on strings unless `OMIT QUOTES`. https://www.postgresql.org/docs/18/functions-json.html
- If the input is text cast implicitly to jsonb, parse errors are raised whatever the ON ERROR clause. https://www.postgresql.org/docs/18/functions-json.html
- `JSON_TABLE` turns a document into rows, joined laterally to its source row. Sibling `NESTED PATH` entries are unioned, not crossed. https://www.postgresql.org/docs/18/functions-json.html
  ```sql
  SELECT jt.* FROM my_films,
   JSON_TABLE (js, '$.favorites[*]' COLUMNS (
     id FOR ORDINALITY,
     kind text PATH '$.kind',
     title text PATH '$.films[*].title' WITH WRAPPER)) AS jt;
  ```
- PostgreSQL 18: `jsonb_populate_record_valid(base, json)` tells whether `jsonb_populate_record` would succeed, without raising. https://www.postgresql.org/docs/18/functions-json.html
- `jsonb_populate_record` and `jsonb_to_record` ignore unknown keys and set missing columns to NULL. Call them in FROM (lateral) to avoid repeated calls. https://www.postgresql.org/docs/18/functions-json.html
- Validate text with `js IS JSON OBJECT` (or `ARRAY`, `SCALAR`, `WITH UNIQUE KEYS`). https://www.postgresql.org/docs/18/functions-json.html

## Building and aggregating JSON

- `json_object(... ABSENT ON NULL)` drops NULL values. The default keeps them as JSON null. SQL/JSON constructors return `json` unless `RETURNING jsonb`. https://www.postgresql.org/docs/18/functions-json.html
- `jsonb_build_object` turns non-text keys into strings silently. https://www.postgresql.org/docs/18/functions-json.html
- `jsonb_agg` keeps NULLs, `jsonb_agg_strict` skips them. `jsonb_object_agg_unique` errors on duplicate keys. Put `ORDER BY` inside the call for a stable order. An aggregate over zero rows returns NULL. https://www.postgresql.org/docs/18/functions-aggregate.html
