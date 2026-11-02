# Ranking and sorting

Default ranking, `sort_by` rules, text match scoring, blending relevance with popularity or recency, conditional sorts, decay functions and pinning in Typesense 30.2.

## Contents

- Default order
- sort_by rules
- Text match score
- Relevance plus popularity: buckets
- Conditional sorts with _eval
- Random and decay sorts
- Missing values
- Pinning and hiding
- Old patterns

## Default order

- Without `sort_by`: `_text_match:desc,default_sorting_field:desc`, then insertion order.
- `default_sorting_field` is set in the schema and must be `int32` or `float`.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/ranking-and-relevance.html

## sort_by rules

- Up to 3 sort fields, later ones break ties. With 1 or 2 fields, `_text_match` is appended as the last tie-breaker. The ranking guide says "up to two" user fields after `_text_match`.
- A hard sort on a field puts `_text_match` last: `price:desc,_text_match:desc`.
- Numeric and boolean fields sort by default. A string field needs `"sort": true` in the schema, which builds a memory-heavy index. Enable it only where needed.
- Range faceting also needs the field's `sort` property.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/ranking-and-relevance.html

## Text match score

- The per-field score combines token frequency, edit distance, proximity, `query_by` order and `query_by_weights`.
- `text_match_type`:
  - `max_score` (default): the best field score represents the document, weights only break ties.
  - `max_weight`: the highest-weighted matching field wins, so a partial match on a heavy field beats a full match on a light one.
  - `sum_score`: weighted sum, so partial matches on many light fields can beat one full match on a heavy field.
- `prioritize_exact_match` (default `true`) ranks a value identical to the query first.
- `prioritize_token_position` (default `false`) ranks earlier occurrences higher.
- `prioritize_num_matching_fields` (default `true`) ranks documents matching more fields higher.

https://typesense.org/docs/guide/ranking-and-relevance.html and https://typesense.org/docs/30.2/api/search.html

## Relevance plus popularity: buckets

A plain `popularity:desc` drowns relevance. Default: bucket the text match so documents tie, and let the business field order each bucket.

```json
{ "sort_by": "_text_match(buckets: 10):desc,weighted_score:desc" }
{ "sort_by": "_text_match(bucket_size: 3):desc,weighted_score:desc" }
```
https://typesense.org/docs/guide/ranking-and-relevance.html

- `buckets` fixes the number of relevance groups. `bucket_size` fixes the results per group, `0` disables, a size above the result count means no bucketing.
- More buckets or a smaller `bucket_size` keeps more relevance and gives less weight to the second field.
- Recency: an `int64` Unix timestamp, `_text_match(buckets: 10):desc,published_at_timestamp:desc`.
- The popularity value can come from the app or from a `counter` analytics rule.
- Per user group: one popularity field per group (`hobbyist_plan_popularity_score`), picked at query time.

https://typesense.org/docs/guide/ranking-and-relevance.html, https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/personalization.html

## Conditional sorts with _eval

- Boost matching documents without filtering others out: `_eval(in_stock:true):desc,popularity:desc`. The expression uses `filter_by` syntax, `&&` and geo filters.
- Score several groups: `_eval([ (brand:Nike):3, (brand:Adidas):2 ]):desc`.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/ranking-and-relevance.html

## Random and decay sorts

- `_rand(42)` gives a stable order for a fixed seed (A/B tests, sampling). `_rand()` seeds from the timestamp. The seed is a positive integer. Combine: `_rand():desc,popularity:desc`.
- Decay: `field(origin: V, func: gauss|linear|exp|diff, scale: N, decay: 0.5, offset: 0):desc`. `origin` (integer) and `scale` (non-zero integer) are required. `decay` lies between 0.0 and 1.0, lower means faster decay.

```
sort_by=timestamp(origin: 1728385250, func: gauss, scale: 1000, decay: 0.5):desc
```
- `func: diff` with `asc` puts values closest to the origin first. `offset` ignores values within that radius.

https://typesense.org/docs/30.2/api/search.html

## Missing values

- Optional numeric fields that are null or missing sort LAST in both directions.
- Optional string fields that are empty, null or missing count as the HIGHEST value: last on `asc`, first on `desc`.
- Override per field: `sort_by=title(missing_values: first):asc` or `last`.

https://typesense.org/docs/30.2/api/search.html

## Pinning and hiding

- `pinned_hits=123:1,456:5` (id:position) and `hidden_hits=123,456`. Curation sets apply first, then `pinned_hits`, then `hidden_hits`.
- Pinned and curated hits ignore `filter_by` unless `filter_curated_hits: true`.
- Merchandising: keep a category page to `pinned_hits` mapping in a CMS and pass it at render time.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/ranking-and-relevance.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Algolia `customRanking`, replica index per sort | `sort_by=_text_match(buckets: 10):desc,f1:desc,f2:desc` on one collection | https://typesense.org/docs/guide/migrating-from-algolia.html |
| Algolia optional filters | `_eval(...)` in `sort_by` | https://typesense.org/docs/guide/migrating-from-algolia.html |
| Re-ranking popularity in the app | bucketed `_text_match` | https://typesense.org/docs/guide/ranking-and-relevance.html |
