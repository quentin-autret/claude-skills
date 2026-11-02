# Strings, pattern matching, full-text search

## Contents
- String functions
- Quoting for dynamic SQL
- LIKE, SIMILAR TO, regular expressions
- Full-text search: documents and queries
- Full-text search: indexes
- Ranking and highlighting

## String functions

- `length()` counts characters, `octet_length()` bytes (`octet_length('josé')` is 5 in UTF8). https://www.postgresql.org/docs/18/functions-string.html
- `trim(... 'xy' ...)` removes any of a set of characters, not a substring. `lpad` and `rpad` truncate strings already longer than the target. https://www.postgresql.org/docs/18/functions-string.html
- Prefix test: `starts_with(s, prefix)` or `s ^@ prefix`. https://www.postgresql.org/docs/18/functions-string.html
- `split_part(s, ',', -1)` counts from the end. `left(s, -2)` drops the last 2 characters. https://www.postgresql.org/docs/18/functions-string.html
- `string_to_array(s, NULL)` splits into characters. `array_to_string` drops NULL elements unless you pass a null string. https://www.postgresql.org/docs/18/functions-string.html https://www.postgresql.org/docs/18/functions-array.html
- PostgreSQL 18 `casefold()` folds case for case-insensitive matching (UTF8 only, may change length). Under libc it equals `lower()`. https://www.postgresql.org/docs/18/functions-string.html
- `text || array` is not covered by the `text || anynonarray` operator (it would be ambiguous with the array `||` operators): cast the array to text explicitly. https://www.postgresql.org/docs/18/functions-string.html

## Quoting for dynamic SQL

- Build statements with `format()`: `%I` quotes identifiers (errors on NULL), `%L` quotes literals and writes NULL unquoted, `%s` inserts raw text. https://www.postgresql.org/docs/18/functions-string.html
  ```sql
  SELECT format('INSERT INTO %I VALUES(%L)', 'Foo bar', E'O\'Reilly');
  -- INSERT INTO "Foo bar" VALUES('O''Reilly')
  ```
- `quote_literal(NULL)` returns NULL and nulls the whole string: use `quote_nullable`. https://www.postgresql.org/docs/18/functions-string.html

## LIKE, SIMILAR TO, regular expressions

- LIKE and SIMILAR TO match the whole string. `~` matches anywhere unless anchored with `^` and `$`. https://www.postgresql.org/docs/18/functions-matching.html
- Escape `_` and `%` in LIKE with a backslash (the default ESCAPE) when the pattern comes from user input. https://www.postgresql.org/docs/18/functions-matching.html
- A regex from an untrusted source can take arbitrary amounts of time and memory. Prefer LIKE, prefix the pattern with `***=` to make it literal, or set a `statement_timeout`. https://www.postgresql.org/docs/18/functions-matching.html
- Nondeterministic (case-insensitive) collations work with LIKE, not with ILIKE, SIMILAR TO or regex. https://www.postgresql.org/docs/18/functions-matching.html
- Pick one function: `regexp_like(s, p)` for a test, `regexp_substr` for the whole match (NULL if none), `regexp_match` for capture groups (text array), `regexp_matches(..., 'g')` for every match as rows. https://www.postgresql.org/docs/18/functions-matching.html
  ```sql
  SELECT (regexp_match('foobarbequebaz', 'bar.*que'))[1];
  ```
- `regexp_matches` with no match returns no row, which drops the row from the query. Use `regexp_match` to get NULL instead. https://www.postgresql.org/docs/18/functions-matching.html
- `substring(s FROM 'o(.)b')` returns the first capture group, not the match. Use `(?:...)` for non-capturing groups. https://www.postgresql.org/docs/18/functions-matching.html
- `regexp_replace` replaces only the first match without the `g` flag. https://www.postgresql.org/docs/18/functions-matching.html
  ```sql
  regexp_replace('foobarbaz', 'b(..)', 'X\1Y', 'g')   -- fooXarYXazY
  ```
- In the C locale, `\w`, `[[:alpha:]]` and similar classes never match non-ASCII characters. Word boundaries: `\m`, `\M`, `\y`. https://www.postgresql.org/docs/18/functions-matching.html
- A branch takes the greediness of its first quantified atom, and a regex with a top-level `|` is always greedy, so mixing greedy and non-greedy parts can surprise. https://www.postgresql.org/docs/18/functions-matching.html

## Full-text search: documents and queries

- Do not search natural language with LIKE or regex: no stemming, no ranking, no index. https://www.postgresql.org/docs/18/textsearch-intro.html
- Always normalize with `to_tsvector` and a query function. A plain `::tsvector` cast does not stem: `'fat cats ate fat rats'::tsvector @@ to_tsquery('fat & rat')` is false. https://www.postgresql.org/docs/18/textsearch-intro.html
- Parse raw user input with `websearch_to_tsquery`, which never raises syntax errors and supports quotes, `or` and `-`. `to_tsquery` needs valid operator syntax. https://www.postgresql.org/docs/18/textsearch-controls.html
  ```sql
  SELECT websearch_to_tsquery('english', '"supernovae stars" -crab');
  ```
- `coalesce` every field: one NULL makes the whole document NULL. https://www.postgresql.org/docs/18/textsearch-controls.html
- tsquery precedence: `!` binds tightest, then `<->`, `&`, `|`. https://www.postgresql.org/docs/18/textsearch-intro.html

## Full-text search: indexes

- Default: a stored generated `tsvector` column with a GIN index. Queries need not name the configuration and rechecks are cheaper. https://www.postgresql.org/docs/18/textsearch-tables.html
  ```sql
  ALTER TABLE pgweb
      ADD COLUMN textsearchable_index_col tsvector
          GENERATED ALWAYS AS (to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))) STORED;
  CREATE INDEX textsearch_idx ON pgweb USING GIN (textsearchable_index_col);
  ```
- Escape hatch, an expression index: it must use the 2-argument `to_tsvector('english', body)`, and queries must repeat that exact form to use it. https://www.postgresql.org/docs/18/textsearch-tables.html
  ```sql
  CREATE INDEX pgweb_idx ON pgweb USING GIN (to_tsvector('english', body));
  ```
- GIN is the preferred text search index. GiST is lossy and needs heap rechecks. https://www.postgresql.org/docs/18/textsearch-indexes.html
- With ICU or builtin collation providers, PostgreSQL 18 reads text search files with the cluster's provider: reindex full-text and `pg_trgm` indexes after `pg_upgrade`. https://www.postgresql.org/docs/18/release-18.html

## Ranking and highlighting

- Weight fields with `setweight(..., 'A')` to `'D'` and join with `||`. https://www.postgresql.org/docs/18/textsearch-controls.html
  ```sql
  SELECT title, ts_rank_cd(textsearch, query) AS rank
  FROM apod, to_tsquery('neutrino|(dark & matter)') query
  WHERE query @@ textsearch
  ORDER BY rank DESC
  LIMIT 10;
  ```
- Ranking consults the tsvector of each matching document and can be I/O bound. `ts_headline` uses the original document, not a tsvector, so it can be slow: use with care. https://www.postgresql.org/docs/18/textsearch-controls.html
- `ts_headline` output is not HTML-safe: sanitize it or strip HTML from input. https://www.postgresql.org/docs/18/textsearch-controls.html
- Limits: tsvector under 1MB, positions up to 16383, lexemes under 2KB. https://www.postgresql.org/docs/18/textsearch-limitations.html
