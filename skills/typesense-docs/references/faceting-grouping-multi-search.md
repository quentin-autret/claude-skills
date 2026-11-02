# Faceting, grouping and multi-search

Facets and their strategies, `group_by`, federated and union multi-search in Typesense 30.2.

## Contents

- Faceting
- Facet counts for filter widgets
- Grouping
- Federated multi-search
- Union multi-search
- Old patterns

## Faceting

- `facet_by` needs `"facet": true` on the field in the schema. Filtering does not.
- Integer facets also return min, max, sum and avg.
- `max_facet_values` defaults to 10.
- Sort facet values: `phone(sort_by: _alpha:asc)` or by a sibling field `recipe.name(sort_by: recipe.calories:asc)`.
- Numeric ranges: `rating(Average:[0, 3], Good:[3, 4], Great:[4, 5])`. Start inclusive, end exclusive, a blank bound means min or max: `price(economy:[0, 100], others:[100, ])`.
- `facet_query` searches facet values by prefix and highlights them. With `facet_query`, `per_page` defaults to 0 (facets only) unless set. `facet_query_num_typos` defaults to 2.
- `facet_strategy`: `automatic` (default), `exhaustive` for small result sets or many requested values, `top_values` for huge hit counts (then `total_values` is not exact).
- Sampling for speed: `facet_sample_percent` (default 100, off), `facet_sample_threshold` (default 0). Counts become estimates.
- `facet_return_parent` returns the parent object of nested facet fields, wildcards allowed (`product.color.*`).
- Joined facets: `facet_by=$Customers(product_price)` with `filter_by=$Customers(customer_id: customer_a)`.

https://typesense.org/docs/30.2/api/search.html

```js
client.collections('products').documents().search({
  q: '*', query_by: 'name',
  facet_by: 'brand,rating(Average:[0, 3], Good:[3, 4], Great:[4, 5])',
  max_facet_values: 20
})
```
https://typesense.org/docs/30.2/api/search.html

## Facet counts for filter widgets

Once a filter applies, facet counts shrink to the filtered value. To show counts of unselected values, send one extra query without that filter per filter widget, all in one `multi_search`.
https://typesense.org/docs/guide/faqs.html

## Grouping

- `group_by` fields MUST be facet fields. `group_limit` defaults to 3 (server cap 99).
- With `group_by`, `per_page` counts groups, not hits.
- Responses use `grouped_hits` (each with `group_key` and nested `hits`), not the flat `hits` array.
- `found` is approximate with `group_by`. `group_max_candidates` makes it exact up to its value.
- `group_missing_values` defaults to `true`: all null values form one group.
- Sort groups by size: `sort_by=_group_found:desc`.
- Dedupe: `group_by=size&group_limit=1`. Limit skew: `group_by=brand&group_limit=3`.

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/30.2/api/server-configuration.html

## Federated multi-search

- `POST /multi_search` runs many searches in one HTTP request. `results` come in the same order as `searches`. Shared parameters go in the second argument.
- `limit_multi_searches` (default 50) caps searches per request. Embed it in a scoped search key.
- Each search can carry its own `x-typesense-api-key`, for a different scoped search key per collection.
- `use_cache` goes in the URL query string, not the body.
- The docs send vector queries (large query vectors) through `multi_search` as a POST body. Image and voice query examples also use `multi_search`.

```js
client.multiSearch.perform({
  searches: [
    { collection: 'products', q: 'shoe', filter_by: 'price:=[50..120]' },
    { collection: 'brands', q: 'Nike' }
  ]
}, { query_by: 'name' })
```
https://typesense.org/docs/30.2/api/federated-multi-search.html, https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/30.2/api/vector-search.html

## Union multi-search

- `union: true` merges every search into ONE ranked list, across collections.
- Per-search `page`, `per_page`, `offset`, `limit` and `limit_hits` are ignored. Pass them globally as URL query parameters: `/multi_search?page=1&per_page=2`.
- Every search needs the same type, count and order of sort fields. `user_name:asc` in one and `rating:asc` in another is an error.
- Duplicates are removed by default, `remove_duplicates: false` keeps them.
- With `group_by`, either all searches group or none.

```js
await client.multiSearch.perform(
  { union: true, remove_duplicates: true, searches: queries },
  { page, per_page: 10 }
)
```
https://typesense.org/docs/30.2/api/federated-multi-search.html and https://typesense.org/docs/guide/boolean-tag-search.html

Boolean tag search with typo-tolerant tags: autocomplete tags with a normal `q` search, build `filter_by` from the chosen tags (OR inside a field, AND between fields, `:=` for strings, `:` for arrays) with `drop_tokens_threshold: 0`, and merge free-text tags as separate searches with `union: true`.
https://typesense.org/docs/guide/boolean-tag-search.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| One collection with every field `optional` plus `document_type` for a cross-collection list | `union: true` in `multi_search` | https://typesense.org/docs/guide/faqs.html |
| Exact `found` with `group_by` | approximate `found`, `group_max_candidates` for accuracy | https://typesense.org/docs/30.2/api/search.html |
| Algolia `distinct` | `group_by` + `group_limit` | https://typesense.org/docs/guide/migrating-from-algolia.html |
