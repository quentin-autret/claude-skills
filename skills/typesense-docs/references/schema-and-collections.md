# Schema and collections

Field definitions, memory cost, auto schema detection, nested objects, altering a collection, aliases and collection layout in Typesense 30.2.

## Contents

- Memory rule: what to put in the schema
- Field parameters and defaults
- Field types
- Auto schema detection
- Nested objects
- Altering a collection
- Aliases and zero-downtime schema changes
- Clone, truncate, drop, metadata
- Laying out collections
- Old patterns

## Memory rule: what to put in the schema

- Every field named in the schema is indexed in memory. A field that is only displayed (an image URL, a link) stays out of the schema, or gets `index: false`. Such fields are stored on disk only, cost no RAM and are still returned in hits.
  https://typesense.org/docs/30.2/api/collections.html and https://typesense.org/docs/guide/running-in-production.html
- Documents may carry fields that are not in the schema. They are stored and returned, never indexed and never type-checked.
  https://typesense.org/docs/30.2/api/collections.html
- Prefer a pre-defined schema. Auto schema detection can index fields you never search into RAM, unless you set `index: false` on them.
  https://typesense.org/docs/guide/running-in-production.html
- With a pre-defined schema, a document whose types do not match is rejected. The `dirty_values` write parameter changes that.
  https://typesense.org/docs/30.2/api/collections.html

```js
client.collections().create({
  name: 'companies',
  fields: [
    { name: 'company_name', type: 'string' },
    { name: 'num_employees', type: 'int32' },
    { name: 'country', type: 'string', facet: true }
  ],
  default_sorting_field: 'num_employees'
})
```
https://typesense.org/docs/30.2/api/collections.html

## Field parameters and defaults

| Parameter | Default | Note |
| --- | --- | --- |
| `facet` | `false` | faceted values are indexed verbatim, no tokenization |
| `optional` | `false` | a required field must be present in every document |
| `index` | `true` | `false` keeps it out of every in-memory index (search, sort, filter, facet) |
| `store` | `true` | `false` strips the value before it is saved to disk |
| `sort` | `true` for numbers, `false` otherwise | string sorting builds an extra, memory-heavy index |
| `infix` | `false` | "significant memory overhead" |
| `locale` | `en` | language-specific tokenization |
| `num_dim` | none | non-zero turns a `float[]` into a vector field |
| `vec_dist` | `cosine` | or `ip` |
| `reference` | none | JOIN target `collection.field` |
| `range_index` | `false` | faster numeric range filters, extra memory |
| `stem` | `false` | stem values before indexing |
| `truncate_len` | `100` | max characters indexed per token |

https://typesense.org/docs/30.2/api/collections.html

- The v30 release notes call the token truncation parameter `truncate`, the collections page `truncate_len`. Check the running server before relying on either name.
  https://typesense.org/docs/30.2/api/ and https://typesense.org/docs/30.2/api/collections.html
- Collection-level parameters: `token_separators`, `symbols_to_index`, `default_sorting_field` (an `int32` or `float` field), `synonym_sets`, `curation_sets`, `enable_nested_fields`, `metadata` (free object returned by `GET /collections`).
  https://typesense.org/docs/30.2/api/collections.html
- `default_sorting_field` also decides which prefix and typo candidates are the "top" ones. Only the top `max_candidates` (default 4) are searched. Without it, "top" means the variants with the most matching records. With a popularity field, it means the most popular terms.
  https://typesense.org/docs/30.2/api/collections.html

## Field types

`string`, `string[]`, `int32`, `int32[]`, `int64`, `int64[]`, `float`, `float[]`, `bool`, `bool[]`, `geopoint`, `geopoint[]`, `geopolygon`, `object`, `object[]`, `string*`, `image`, `auto`.
https://typesense.org/docs/30.2/api/collections.html

- `int32` stops at 2,147,483,647. Use `int64` above.
- There is no date type. Store dates as Unix timestamps in `int64` and filter with `<` and `>`.
- `geopoint` is `[lat, lng]` and always needs an explicit definition, because auto detection cannot tell it from a float array.
- `string*` converts any value to `string` or `string[]`.
- `image` is a base64 image string for image search.

https://typesense.org/docs/30.2/api/collections.html and https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html

## Auto schema detection

```json
{ "name": "companies", "fields": [
  {"name": ".*", "type": "auto"},
  {"name": ".*_facet", "type": "auto", "facet": true},
  {"name": "description_.*", "type": "string", "index": false, "optional": true},
  {"name": "country", "type": "string", "facet": true}
] }
```
https://typesense.org/docs/30.2/api/collections.html

- Any RegEx works as a field name. The first document holding a field fixes its type (`4200` becomes `int64`). Later values are coerced when possible, otherwise the write fails.
- An explicit field definition wins over the wildcard.
- Excluding a field from indexing needs `index: false` with `optional: true`, since a mandatory field cannot be excluded.
- A wildcard `.*` field never gets `facet` implicitly. Set `facet: true` on named or RegEx-named fields.
- Fields added on the fly: use RegEx field names. New matching fields join the schema automatically.

https://typesense.org/docs/30.2/api/collections.html

## Nested objects

- `enable_nested_fields: true` on the collection, then `object` or `object[]` types.
- Querying an object field searches all its sub-fields. Target one with dot notation, `person.last_name`.
- Index only some sub-fields by naming them: `{"name": "address.zip", "type": "string"}`. Under an `object[]`, sub-fields are arrays: `{"name": "addresses.zip", "type": "string[]"}`.
- When definitions overlap at different levels, the broader one wins over the sub-field one.

https://typesense.org/docs/30.2/api/collections.html

## Altering a collection

```js
client.collections('companies').update({
  fields: [
    { name: 'company_category', drop: true },
    { name: 'company_category', type: 'string', facet: true }
  ]
})
```
https://typesense.org/docs/30.2/api/collections.html

- Only add and drop exist. Changing a field is a drop plus an add of the same name in one request. Every field can change except `id`.
- An alter is synchronous and blocks all writes to that collection, on every node of an HA cluster. Reads continue. Alter one field at a time on large collections, off-peak.
- One alter at a time per cluster. The call returns only when done, so give the client a long timeout instead of retrying.
- The alter validates stored documents first. Adding `A` as integer is rejected when stored documents hold `A` as strings.
- Dropping a field only removes it from the schema and memory. To erase the data, update documents with `null` for that field.
- Progress: `GET /operations/schema_changes` returns `[{"collection":"companies","validated_docs":873000,"altered_docs":0}]`, empty when idle.

https://typesense.org/docs/30.2/api/collections.html

## Aliases and zero-downtime schema changes

An alias is a virtual collection name, usable in every call that takes a collection name. The application only uses the alias.
https://typesense.org/docs/30.2/api/collection-alias.html

1. Alias `movies` points to `movies_jan_1`.
2. Create `movies_feb_1` with the new schema.
3. Dual-write to both collections.
4. Upsert all data from the primary database into the new one.
5. Repoint the alias.
6. Stop writing to the old collection and drop it.

https://typesense.org/docs/30.2/api/collections.html

```js
client.aliases().upsert('companies', { collection_name: 'companies_june11' }) // PUT /aliases/:alias
client.aliases('companies').retrieve()
client.aliases('companies').delete()
```
https://typesense.org/docs/30.2/api/collection-alias.html

## Clone, truncate, drop, metadata

- Truncate keeps the schema: `documents().delete({ truncate: true })` returns `{"num_deleted": N}`.
- Clone the schema, overrides and synonyms: `POST /collections?src_name=existing` with `{"name": "new_coll"}`. Add `copy_documents=true` to copy documents too.
- Dropping a collection triggers disk compaction. The `compact_store` GET parameter disables it when you drop often.
- Listing returns all collections, newest first. Paginate with `offset` and `limit`, and `exclude_fields=fields` omits field definitions.

https://typesense.org/docs/30.2/api/collections.html

## Laying out collections

- One collection per document type. Unrelated products still share one collection when their specific fields are `optional`.
- Multi-tenant: one collection with a tenant or user id field, plus one scoped search key per user with an embedded filter.
- Sharding: when search slows, split by an attribute (`users_usa`) or `user_id mod n`. Move a very large tenant into its own collection.
- Environments: one cluster each, or one cluster with suffixed collections (`x_production`, `x_staging`) and separate keys per environment.

https://typesense.org/docs/guide/organizing-collections.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Flattening objects into dotted keys | `enable_nested_fields` with `object` types | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |
| Synonyms nested under a collection | top-level synonym sets linked by `synonym_sets` | https://typesense.org/docs/30.2/api/ |
| Overrides nested under a collection | top-level curation sets linked by `curation_sets` | https://typesense.org/docs/30.2/api/ |
| Collection-level `token_separators` only | field-level `token_separators` and `symbols_to_index`, which win | https://typesense.org/docs/30.2/api/collections.html |
| Clone copying the schema only | `copy_documents=true` also copies documents | https://typesense.org/docs/30.2/api/collections.html |
