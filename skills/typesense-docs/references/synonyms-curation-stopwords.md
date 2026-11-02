# Synonym sets, curation sets and stopwords

Top-level synonym sets and curation sets, their key actions, tags and multi-tenant pitfalls, plus stopword sets in Typesense 30.2.

## Contents

- Synonym sets
- Curation sets
- Curation tags
- Dynamic filtering and sorting
- Curation and scoped search keys
- Diversity
- Stopwords
- Old patterns

## Synonym sets

- Synonyms live in top-level synonym sets at `/synonym_sets`. A set does nothing until it is linked to a collection (`synonym_sets` array at creation or by `PATCH`) or named in the `synonym_sets` search parameter. Both sources apply together.
- Multi-way: every word finds the others. One-way: `root` finds the synonyms, not the reverse.

```js
client.synonymSets('clothing-synonyms').upsert({
  items: [{ id: 'coat-synonyms', synonyms: ['blazer', 'coat', 'jacket'] }]
})
client.synonymSets('tech-synonyms').upsert({
  items: [{ id: 'smart-phone-synonyms', root: 'smart phone', synonyms: ['iphone', 'android'] }]
})
```
```bash
curl "http://localhost:8108/collections/products" -X PATCH -H "Content-Type: application/json" \
  -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" -d '{"synonym_sets": ["clothing-synonyms", "tech-synonyms"]}'
```
https://typesense.org/docs/30.2/api/synonyms.html

- Item fields: `id` and `synonyms` required, `root` for one-way, `locale` (applies only when the most weighted queried field has that locale), `symbols_to_index`.
- Item calls: `synonymSets(name).items(id).upsert / retrieve / delete`.
- Synonyms apply to `q` tokens only: never in phrase search (double quotes), never to `filter_by`.
- Curation runs first and can rewrite the query. Synonyms apply to the rewritten query.
- Search parameters: `enable_synonyms=false`, `synonym_prefix` (default `false`), `demote_synonym_match=true` ranks synonym matches below direct ones, `synonym_num_typos` (default 0).

https://typesense.org/docs/30.2/api/synonyms.html and https://typesense.org/docs/30.2/api/search.html

## Curation sets

- Curation rules (pin, hide, filter, sort per query) live in top-level curation sets at `/curation_sets`, linked by the collection `curation_sets` array.

```js
client.curationSets().upsert('curate_products', { items: [{
  id: 'customize-apple',
  rule: { query: 'apple', match: 'exact' },
  includes: [{ id: '422', position: 1 }, { id: '54', position: 2 }],
  excludes: [{ id: '287' }]
}]})
```
https://typesense.org/docs/30.2/api/curation.html

- `rule` needs `query` or `filter_by`. `match` (`exact` or `contains`) is required with `query`.
- `rule.filter_by` must equal the search `filter_by` EXACTLY, backticks, spaces and brackets included.
- Items apply in lexical order of `id`. `stop_processing` defaults to `true`, so only the first matching rule applies. Set `false` to chain rules.
- Other item parameters: `metadata` (returned in the response, for a promo banner), `filter_by`, `sort_by`, `replace_query`, `remove_matched_tokens` (default `true`), `filter_curated_hits` (default `false`), `effective_from_ts` and `effective_to_ts` (Unix timestamps), `rule.synonyms` and `rule.stem` (default `false`).
- Pinned `includes` ignore `filter_by` unless `filter_curated_hits: true`, then the remaining pinned hits move up.
- `enable_curations=false` disables curation for one query.

https://typesense.org/docs/30.2/api/curation.html and https://typesense.org/docs/30.2/api/search.html

## Curation tags

- `curation_tags=tagA`: only items tagged `tagA` are considered.
- `curation_tags=tagA,tagB`: items with both tags first, then items with either, never untagged items.
- No `curation_tags`: ONLY untagged items are considered. A tagged rule never fires unless the search sends its tag.

https://typesense.org/docs/30.2/api/curation.html

## Dynamic filtering and sorting

```json
{ "id": "brand-phone",
  "rule": { "query": "{brand} phone", "match": "contains" },
  "filter_by": "brand:={brand}", "remove_matched_tokens": true }
```
- Placeholder fields need `facet: true`.
- Dynamic sort: `"sort_by": "sales.{store}:desc, inventory.{store}:desc"` with `rule.query: "{store}"` or `rule.filter_by: "store:={store}"`.

https://typesense.org/docs/30.2/api/curation.html

## Curation and scoped search keys

- With a `filter_by` embedded in a scoped search key, the final filter is `(request_filter) && (scoped_api_key_filter)`. A filter-based rule must match that string, for example `(title:hold) && (country: AU)`.
- Curated hits are not filtered by the key's embedded filter either. In multi-tenant setups, embed `{"filter_by": "company_id:124", "filter_curated_hits": true}` in the key, or a pinned document from another tenant can leak.

https://typesense.org/docs/30.2/api/curation.html

## Diversity

- Item `diversity.similarity_metric`: a list of `{field, method, weight}`, `method` among `equality`, `jaccard` (arrays only) and `vector_distance`. The field needs `facet: true` for arrays, `sort: true` otherwise. Diversity works on the top 250 hits.
- Request parameter `diversity_lambda` (default 0.5): 1 means no diversity, 0 maximum diversity.

https://typesense.org/docs/30.2/api/curation.html

## Stopwords

- Stopwords are removed from the query only, never at indexing.
- `PUT /stopwords/<set>` with `{"stopwords": [...], "locale": "en"}`. A `PUT` on an existing set replaces it.
- Stopwords are tokenized and lowercased: "United States" is stored as `united` and `states`, each removed on its own.
- Apply with the `stopwords=<set>` search parameter.
- The multi-search example on the stopwords page has syntax errors (`query_b`, missing commas). Do not copy it.

https://typesense.org/docs/30.2/api/stopwords.html

## Old patterns

Synonyms and overrides were migrated automatically on upgrade to v30, into `<collection>_synonyms_index` and `<collection>_curations_index`. Self-hosted servers need a snapshot BEFORE upgrading, or the migration may be lost. Keys cannot be updated, so create new keys and delete the old ones.

| Old | Current | Source |
| --- | --- | --- |
| `/collections/{c}/synonyms` | `/synonym_sets` + collection `synonym_sets` | https://typesense.org/docs/30.2/api/synonyms.html |
| `synonyms:*` key actions (now `401`) | `synonym_sets:*` and `synonym_sets/items:*` | https://typesense.org/docs/30.2/api/synonyms.html |
| `/collections/{c}/overrides` | `/curation_sets` + collection `curation_sets` | https://typesense.org/docs/30.2/api/curation.html |
| `overrides:*` key actions (now `401`) | `curation_sets:*` and `curation_sets/items:*` | https://typesense.org/docs/30.2/api/curation.html |
| `override_tags` | `curation_tags` | https://typesense.org/docs/30.2/api/ |
