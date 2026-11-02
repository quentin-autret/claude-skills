# Joins and personalization

Reference fields, joined includes, filters and sorts, left and anti joins, async references, alias pitfalls and personalized pricing by join in Typesense 30.2.

## Contents

- Reference fields
- Including joined fields
- Filtering and sorting through joins
- Inner, left and anti joins
- Async references and cascade delete
- Aliases and joins
- Security
- Personalized search by join
- Old patterns

## Reference fields

```json
{ "name": "books", "fields": [
  {"name": "title", "type": "string"},
  {"name": "author_id", "type": "string", "reference": "authors.id"} ] }
```
https://typesense.org/docs/30.2/api/joins.html

- Declare `reference` fields when the collection is created. Adding one to an existing collection by alter is not supported.
- Types: `int32`, `int64`, `string` for one-to-one, their array forms for one-to-many.
- Many-to-many: a link collection with two reference fields, for example `user_doc_access` referencing `users.id` and `documents.id`, queried with `filter_by: '$user_doc_access(user_id:=user_a)'`.
- Inside objects: `{"name": "order.product_id", "type": "string", "reference": "products.id"}`. Under `object[]` the reference is an array type: `{"name": "orders.product_id", "type": "string[]", "reference": "products.id"}`.
- Two collections referencing each other are not allowed: the reference is not indexed. A reference to a field of the same collection is an error.

https://typesense.org/docs/30.2/api/joins.html and https://typesense.org/docs/30.2/api/

## Including joined fields

- From the collection holding the reference: `include_fields: '$authors(first_name,last_name)'`, or `$authors(*)`.
- Reverse direction (search authors, get their books): `filter_by: '$books(id: *)'` plus `include_fields: '$books(*)'`. `id:*` matches every joined document.
- Rename: `$users(id) as user_identifier`.
- Shape: default `strategy: nest` (object under the collection name), `strategy: merge` (fields merged into the parent), `strategy: nest_array` (always an array).
- A singular reference returns an object for 1 match and an array for more. Use `nest_array` for a stable frontend shape.
- `$books(*, related_docs_count: foo_related_count)` adds a count field. `$books(*, sort_by: id: desc)` and `$books(*, limit:5)` order and cap joined documents.
- The document GET endpoint also takes `include_fields=$authors(name)`, escaping `$` in a shell.

https://typesense.org/docs/30.2/api/joins.html

## Filtering and sorting through joins

- Filter: `$orders(total_price:<100)`, `$customer_product_prices(customer_id:=customer_a && custom_price:<100)`.
- Sort: `sort_by: '$product_prices(price_net:asc)'`. On a to-many join this syntax picks the lowest (asc) or highest (desc) value per document. Without it, the error is `Multiple references found to sort by on product_prices.price_net`.
- `_eval` works inside: `$Coll( _eval(field_name:foo):desc )`. Nested: `$A( $B(field:desc))`.
- Facet through a join: `facet_by: '$Customers(product_price)'` with a join filter.
- Joins can be nested (`$A( $B( $C(...)))`), shown here with 3 levels:

```json
{ "collection": "products", "q": "shampoo", "query_by": "name",
  "filter_by": "$product_variants(price:<100 && $inventory( $retailers(location:(48.87538726829884, 2.296113163780903,1 km))))",
  "include_fields": "$product_variants(price, $inventory(qty, $retailers(title)))" }
```
https://typesense.org/docs/30.2/api/joins.html and https://typesense.org/docs/30.2/api/search.html

## Inner, left and anti joins

- Join filters are INNER joins: documents without a match disappear.
- Left join: `filter_by: 'id:* || $join_collection(<condition>)'`.
- Anti join: `filter_by: '!$deny_list(user_id:=u11)'` with `exclude_fields: '$deny_list(*)'` so deny-list data is not returned.

https://typesense.org/docs/30.2/api/joins.html and https://typesense.org/docs/guide/personalized-search-join.html

## Async references and cascade delete

- Without `async_reference`, indexing a document whose referenced document does not exist yet fails with `Reference document having '...' not found in the collection '...'`.
- `async_reference: true` indexes it anyway and resolves the link when the referenced document arrives. Use it when import order differs from dependency order.
- By default a document is deleted once ALL documents it references are deleted. `cascade_delete: false` stops that and requires `async_reference: true`.

```json
{"name":"product_id","type":"string","reference":"products.product_id","optional":true,"async_reference":true}
```
https://typesense.org/docs/30.2/api/joins.html

## Aliases and joins

A reference may target an alias, but documents store the INTERNAL sequential ids of the referenced documents. Swapping one collection of a join group through an alias requires reindexing every related collection together.
https://typesense.org/docs/30.2/api/joins.html

## Security

A search-only key scoped to the referencing collection can read the referenced collection's fields through a join, though it cannot search that collection directly.
https://typesense.org/docs/30.2/api/joins.html and https://typesense.org/docs/30.2/api/api-keys.html#search-only-api-key

## Personalized search by join

Keep per-user or per-tier overrides in a collection that references the base one, so prices change without reindexing products.

```json
{ "searches": [ {
  "collection": "products", "q": "wireless headphones", "query_by": "name,description,tags",
  "filter_by": "$user_prices(tier:=VIP && region:=US)",
  "include_fields": "$user_prices(custom_price)"
} ] }
```
https://typesense.org/docs/guide/personalized-search-join.html

- This inner join drops products without an override. Left join: `id:* || $user_prices(...)`. Products without an override then have no joined field, and the app falls back to `base_price`.
- Stable shape: `$user_prices(custom_price,tier, strategy: nest_array)`. Sort by the joined value: `$user_prices(custom_price:asc)`.
- Combine with normal filters: `rating:>=4.0 && category:=electronics && $user_prices(tier:=VIP && region:=US)`. Several joins: `$user_prices(...) && $inventory(warehouse:=US-WEST && in_stock:=true)`.
- Multi-tier fallback (VIP, then GOLD, then base): several searches in one `multi_search`, merged in the app.
- Index only active override records and facet only fields users filter by, since everything indexed lives in RAM.

https://typesense.org/docs/guide/personalized-search-join.html

Without joins: per-group popularity fields chosen at query time, `_text_match(buckets: 10):desc,hobbyist_plan_popularity_score:desc`. Per user: a behavior embedding in `sort_by`, `_text_match:desc,_vector_query(embedding:([...])):asc`.
https://typesense.org/docs/guide/personalization.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Denormalized per-user prices in the product document | override collection joined at query time | https://typesense.org/docs/guide/personalized-search-join.html |
| Reference fields could not be altered | altering reference fields is supported, adding one to an existing collection still is not | https://typesense.org/docs/30.2/api/ and https://typesense.org/docs/30.2/api/joins.html |
