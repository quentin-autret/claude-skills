# Redis Search: indexes, queries and vector search

Schema design, query traps, timeouts, index lifecycle and vector indexes of Redis Search (`FT.*`), Redis Open Source 8.10.

## Contents

- Schema design
- Hash versus JSON indexes
- Query syntax
- Results, timeouts and limits
- Index lifecycle
- Vector indexes and KNN
- Old patterns

## Schema design

- Index only the fields queries use: each indexed field costs memory and slows updates.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/
- Pick the cheapest field type: `TAG` for exact matches (short values like "new", IDs, cities), not `TEXT` or `NUMERIC`. `TEXT` only for full text.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/scalable-query-best-practices/ and https://redis.io/docs/latest/develop/ai/search-and-query/query/exact-match/
- With non-threaded search, `SORTABLE` only on fields used in `SORTBY` (threaded search sets all fields `SORTABLE`). `NOSTEM` on names, URLs, emails. `NOINDEX` on fields only returned. `WEIGHT` to rank a title over a body.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/scalable-query-best-practices/ and https://redis.io/docs/latest/develop/ai/search-and-query/indexing/field-and-type-options/
- One index with several prefixes beats several similar indexes:

```
FT.CREATE my_index ON HASH PREFIX 2 "user:" "order:" SCHEMA name TEXT age NUMERIC status TAG
```
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/
- `WITHSUFFIXTRIE` optimizes contains (`*foo*`) and suffix queries, otherwise a brute-force search runs on the trie. Once any TEXT field has a suffix trie, those queries are disabled on fields without one.
  https://redis.io/docs/latest/develop/ai/search-and-query/indexing/field-and-type-options/

## Hash versus JSON indexes

- Hash TAG fields split on `,` by default. JSON TAG fields have no default separator, so `"a,b"` is one tag and `@category:{Gaming}` misses. Use arrays: `$.colors[*] AS colors TAG`.
  https://redis.io/docs/latest/develop/ai/search-and-query/indexing/
- JSON objects cannot be indexed (FT.CREATE errors): index leaf values. Booleans index as TAG, `null` is ignored.
  https://redis.io/docs/latest/develop/ai/search-and-query/indexing/
- Query JSON fields by their `AS` attribute names, never by JSONPath in the query string. JSONPath is fine in `RETURN` and `LOAD`.
  https://redis.io/docs/latest/develop/ai/search-and-query/indexing/

## Query syntax

- Always send `DIALECT 2` or higher. For production set `search-default-dialect 4`. Dialect 1 is the server default and the old syntax.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/scalable-query-best-practices/ and https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/dev-to-prod-best-practices/
- Tags need braces: `@condition:{new}`. The curly brackets are mandatory for tag queries.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/exact-match/
- Tag values with special characters: wrap them in double quotes under dialect 2 instead of escaping.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/exact-match/ and https://redis.io/docs/latest/develop/whats-new/7-4/
- Numeric ranges go in the query string, `@price:[500 1000]`, `(` for exclusive bounds, rather than `FILTER`, which runs after the query.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/range/
- Phrases need escaped quotes, `@description:\"rough terrain\"`, and cannot start with a stop word. Fuzzy is `@field:(%word%)` without quotes.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/exact-match/ and https://redis.io/docs/latest/develop/ai/search-and-query/query/full-text/
- Parenthesize combined queries: in dialect 2 AND binds tighter than OR.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/combined/
- Prefix wildcards need at least 2 characters and stop after 200 expansions (`search-max-prefix-expansions`): the query continues with the first results acquired. Only the Unicode BMP works for wildcards and fuzzy: emojis match nothing.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ and https://redis.io/docs/latest/develop/ai/search-and-query/query/full-text/
- Geo values are `lon,lat`.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/overview/
- `FT.SEARCH` selects and projects. Grouping and computing go to `FT.AGGREGATE`, where reducers are used inside `GROUPBY ... REDUCE`.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/ and https://redis.io/docs/latest/develop/ai/search-and-query/query/aggregation/

## Results, timeouts and limits

- `FT.SEARCH` returns 10 results by default. Always set `LIMIT` or use a cursor, avoid `LOAD *` and project only needed fields.
  https://redis.io/docs/latest/develop/ai/search-and-query/query/range/ and https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/scalable-query-best-practices/
- The default query timeout is 500 ms and `search-on-timeout RETURN` (the default) returns partial results with no error. Use `FAIL` when partial results must not pass for complete ones. `RETURN_STRICT` (8.10) needs `search-workers > 0`.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ and https://redis.io/docs/latest/develop/whats-new/8-10/
- With `search-workers 0`, 8.10 caps the effective timeout at `search-_max-foreground-timeout-limit` (60000 ms), even for `TIMEOUT 0`.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/
- Set `search-on-oom` to `FAIL` or `RETURN` instead of the default `IGNORE`.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/
- Cursors are capped at 128 per shard (`search-index-cursor-limit`). Close them with `FT.CURSOR DEL` or set `MAXIDLE`.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/
- Profile with `FT.PROFILE`, check parsing with `FT.EXPLAIN`.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/

## Index lifecycle

- `FT.CREATE` indexes existing keys in the background. Wait for `FT.INFO` `num_docs` and `indexing` before sending traffic.
  https://redis.io/docs/latest/develop/ai/search-and-query/indexing/ and https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/
- `FT.ALTER` only adds fields. Default for any other change: create an alias at setup, build a new index, switch with `FT.ALIASUPDATE`, roll back by switching back.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/
- `FT.DROPINDEX ... DD` also deletes the documents.
  https://redis.io/docs/latest/develop/ai/search-and-query/best-practices/index-mgmt-best-practices/
- Updates and deletes leave garbage (an update is delete plus reindex) until GC reclaims it, which slows heavy-churn indexes.
  https://redis.io/docs/latest/develop/ai/search-and-query/administration/gc/

## Vector indexes and KNN

- Index choice: `FLAT` below about 1M vectors or when exactness matters, `HNSW` above, `SVS-VAMANA` (8.2) for memory reduction on Intel. Intel's LVQ/LeanVec optimizations are not in the default Redis Open Source build (it needs a build with Intel SVS support), which uses global 8-bit quantization.
  https://redis.io/docs/latest/develop/ai/search-and-query/vectors/ and https://redis.io/docs/latest/develop/ai/search-and-query/vectors/svs-compression/
- The blob length must match `DIM` and `TYPE`, or indexing fails. `FLOAT16`/`BFLOAT16` halve FLOAT32 memory. In `FT.CREATE`, the vector count counts name and value tokens (6 for three pairs).
  https://redis.io/docs/latest/develop/ai/search-and-query/vectors/ and https://redis.io/docs/latest/develop/ai/search-and-query/indexing/field-and-type-options/
- KNN needs dialect 2, and always `SORTBY` on the distance and `LIMIT 0 k`: default order is document score and default LIMIT is 10.

```
FT.SEARCH documents "(@title:Sports)=>[KNN 10 @doc_embedding $BLOB AS dist]" PARAMS 2 BLOB "<blob>" SORTBY dist LIMIT 0 10 DIALECT 2
```
  https://redis.io/docs/latest/develop/ai/search-and-query/vectors/
- JSON: an array of vectors goes in `$.embeddings[*]`. A static `$.embedding` path holding several vectors fails.
  https://redis.io/docs/latest/develop/ai/search-and-query/indexing/
- COSINE distance is bounded by 2, L2 is not: set `VECTOR_RANGE` radii per metric.
  https://redis.io/docs/latest/develop/ai/search-and-query/vectors/
- Cluster: `$SHARD_K_RATIO` (default 1.0) lowers per-shard candidates. With unbalanced shards a low ratio can return fewer than k results.
  https://redis.io/docs/latest/develop/ai/search-and-query/vectors/
- `FT.HYBRID` (8.4) fuses text and vector scores (RRF or linear). It gained `EXPLAINSCORE` in 8.10, and its `KNN` clause accepts `SHARD_K_RATIO`.
  https://redis.io/docs/latest/develop/whats-new/8-4/ and https://redis.io/docs/latest/develop/whats-new/8-10/ and https://redis.io/docs/latest/develop/ai/search-and-query/vectors/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `BM25` scorer | `BM25STD`, the default since 8.4 | https://redis.io/docs/latest/develop/ai/search-and-query/administration/overview/ |
| `TIMEOUT`, `MINPREFIX`, `MAXSEARCHRESULTS` module args | `search-*` names in `redis.conf` or `CONFIG SET` (8.0) | https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ |
| `MT_MODE`, `WORKER_THREADS` | `search-workers`, `search-min-operation-workers` | https://redis.io/docs/latest/develop/ai/search-and-query/administration/configuration/ |
| Escaping `@` and `.` in tag queries | double quotes (7.4) | https://redis.io/docs/latest/develop/whats-new/7-4/ |