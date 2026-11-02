# Search parameters

Query syntax, `query_by`, typo tolerance, token dropping, prefix and infix, highlighting, pagination, caching and presets in Typesense 30.2.

## Contents

- Query basics
- Per-field parameter lists
- Typo tolerance
- Token dropping
- Prefix, infix and max_candidates
- Highlighting and returned fields
- Pagination and limits
- Caching, presets, cutoffs
- Old patterns

## Query basics

- `q` and `query_by` are required. `q: '*'` returns every document, usually with `filter_by`.
- Phrase search: double quotes in `q`. Exact word order, no typo tolerance.
- Exclude a word: `q: 'electric car -tesla'`.
- Only `string` and `string[]` fields can be in `query_by`. An `object` field expands to its string children.
- Field order in `query_by` matters: a match in an earlier field outranks one in a later field. `query_by_weights` (0 to 127) overrides that order.

```js
client.collections('companies').documents().search({
  q: 'stark', query_by: 'company_name',
  filter_by: 'num_employees:>100', sort_by: 'num_employees:desc'
})
```
https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/ranking-and-relevance.html

## Per-field parameter lists

Several parameters take a comma list in `query_by` order, and a single value applies to every field: `prefix=true,false,false`, `num_typos=2,0,0`, `infix=off,always`.
https://typesense.org/docs/30.2/api/search.html

## Typo tolerance

| Parameter | Default | Note |
| --- | --- | --- |
| `num_typos` | 2 | 0, 1 or 2, Damerau-Levenshtein |
| `min_len_1typo` | 4 | shortest word corrected with 1 typo |
| `min_len_2typo` | 7 | shortest word corrected with 2 typos |
| `typo_tokens_threshold` | 1 | typo variants are searched while fewer results exist, `0` disables |
| `split_join_tokens` | `fallback` | tries `basket ball` and `basketball` only when nothing matched, `always` or `off` |
| `enable_typos_for_numerical_tokens` | `true` | `false` for numbers |
| `enable_typos_for_alpha_numerical_tokens` | `true` | `false` for SKU-like tokens |
| `synonym_num_typos` | 0 | typo-corrected words do not resolve synonyms |

https://typesense.org/docs/30.2/api/search.html

- Turning typos off needs BOTH `num_typos=0` and `typo_tokens_threshold=0`.
- Per-field example for codes: `query_by=name,phone_number,zip_code` with `num_typos=2,0,0`.

https://typesense.org/docs/guide/ranking-and-relevance.html

## Token dropping

- `drop_tokens_threshold` defaults to 1. When fewer results match all words, words are dropped one by one and the search repeats. A query with 0 or 1 hit silently loosens, so an empty page is rare.
- Require every word: `drop_tokens_threshold: 0`.
- `drop_tokens_mode`: `right_to_left` (default), `left_to_right`, `both_sides:3` (exhaustive up to 3 words, then `right_to_left`).
- OR-like search over all words: a huge threshold with `both_sides:3`.
- In hybrid search, multi-word queries should send `drop_tokens_threshold: 0` to avoid redundant keyword passes.

https://typesense.org/docs/30.2/api/search.html, https://typesense.org/docs/guide/ranking-and-relevance.html, https://typesense.org/docs/guide/faqs.html and https://typesense.org/docs/30.2/api/vector-search.html

## Prefix, infix and max_candidates

- `prefix` defaults to `true`: the last query word is a prefix. Disable per field with a list.
- `infix` (`off`, `always`, `fallback`) needs `"infix": true` in the schema and uses only the first query word. `max_extra_prefix` and `max_extra_suffix` bound the characters around the match.
- `max_candidates` defaults to 4: only 4 prefix expansions or typo variants are searched. A short query like "a" can miss expected records. Raise it, or set a popularity `default_sorting_field` so the top candidates are the popular ones.
- Adding a filter can return MORE hits than no filter, because the smaller candidate space is searched more exhaustively. Tune `max_candidates` (for example 100) while watching `search_time_ms`.
- `exhaustive_search=true` considers every variation, ignores both thresholds and raises `max_candidates` to 10000.
- `max_filter_by_candidates` (default 4) bounds the words considered by a prefix filter such as `company_name:Acm*`.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/faqs.html

## Highlighting and returned fields

- `highlight_fields` snippets, `highlight_full_fields` highlights the whole value. `highlight_affix_num_tokens` defaults to 4, tags default to `<mark>`, `snippet_threshold` defaults to 30.
- `highlight_fields=none` disables snippets. `highlight_full_fields=none` disables full highlighting.
- `enable_highlight_v1` defaults to `true` and keeps the old `highlights` array next to the new `highlight` object. Send `false` to drop it.
- `include_fields` and `exclude_fields` take comma lists. Excluding vector fields is usually a good idea (saves bandwidth and CPU).

https://typesense.org/docs/30.2/api/search.html

## Pagination and limits

- `page` and `per_page`, or `offset` and `limit`. At most 250 hits per page (250 groups with `group_by`), the server default `--max-per-page`.
- `limit_hits` caps the total reachable hits. `page * per_page` must stay below it or nothing returns. Embed it in a scoped search key so the client cannot change it.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/30.2/api/server-configuration.html

## Caching, presets, cutoffs

- `use_cache` defaults to `false`. On `multi_search` it must be a URL query parameter, never in the POST body.
- `cache_ttl` (default 60s) can only be set inside a scoped search key.
- `preset` loads stored parameters. Explicit request parameters override the preset. A preset holds either single-search parameters or a `searches` array, and the docs recommend single-search presets.
- `search_cutoff_ms` is a best-effort early return, not a guarantee, and does not bound faceting.
- `validate_field_names` (default `true`) set to `false` stops errors on missing fields, useful for programmatic grouping where not all fields may exist.
- `pre_segmented_query: true` splits the query on spaces only.
- `stopwords` takes stopword set names.
- `enable_lazy_filter: true` applies filtering incrementally, for large filter values when the query tokens match few documents.

https://typesense.org/docs/30.2/api/search.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Response `highlights` array | `highlight` object with `enable_highlight_v1=false` | https://typesense.org/docs/30.2/api/search.html |
| `override_tags` | `curation_tags` | https://typesense.org/docs/30.2/api/ |
