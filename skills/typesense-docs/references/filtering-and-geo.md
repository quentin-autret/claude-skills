# Filtering and geosearch

`filter_by` syntax, operator precedence, escaping, arrays, nested objects, empty values, joins and geo filters in Typesense 30.2.

## Contents

- Syntax
- Strings: `:` versus `:=`
- Lists, ranges, negation
- Precedence and escaping
- Arrays and nested objects
- Empty values
- Filters through joins
- Geosearch
- Old patterns

## Syntax

- The form is `field:<operator><value>`. The colon is mandatory: `price:>=100`, never `price>=100`.
- Operators: `=`, `:`, `>`, `<`, `<=`, `>=`, `!=`, `[]` (one of), `![]` (none of), `[min..max]`. Numeric operators work on `int32`, `int64` and `float`.
- `filter_by` matches exactly (no typo tolerance) and synonyms apply only to `q`, not to `filter_by`. See https://typesense.org/docs/guide/boolean-tag-search.html
- The server caps a filter at 100 operations (`--filter-by-max-ops`).

https://typesense.org/docs/guide/tips-for-filtering.html, https://typesense.org/docs/30.2/api/synonyms.html and https://typesense.org/docs/30.2/api/server-configuration.html

## Strings: `:` versus `:=`

- `:` is a word-level match that ignores token position: `category:Shoe` matches "Shoe Rack" and "Outdoor Shoe". `category:=Shoe` matches only "Shoe".
- `:` is faster. Default to `:` when values never contain spaces, `:=` when the whole value must match.
- `:` matches whole words only: `location:New` finds "New Zealand" but not "Newfoundland".
- Prefix filter: `name:Jo*` matches any word starting with "Jo". `name:=Jo*` matches only values starting with "Jo".

https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/tips-for-filtering.html

## Lists, ranges, negation

```
country:[USA, UK]
num_employees:[<10, >100]
price:[100..200, 15..50, 800]
author:!=JK Rowling
id:!=[id1, id2]
artist:! Jackson
```
https://typesense.org/docs/30.2/api/search.html and https://typesense.org/docs/guide/tips-for-filtering.html

- `artist:! Jackson` excludes documents whose field contains the word.
- Numeric range filters are faster with `"range_index": true` on the field, at a memory cost.

https://typesense.org/docs/30.2/api/search.html

## Precedence and escaping

- `&&` and `||` share the same precedence and evaluate left to right. `country:=USA || country:=Canada && city:=New York` reads as `(USA || Canada) && New York`. Always add parentheses.
- Cross-field OR needs `||`. Same-field OR is a list: `field:[a,b]`.
- Values containing `&&`, `||`, `(`, `)`, `,`, `[` or `]` go between backticks. Field names cannot be escaped.

```
country:=`United States, Minor Outlying Islands`
category:= [`Running Shoes, Men`, `Sneaker (Men)`, Boots]
```
https://typesense.org/docs/guide/tips-for-filtering.html and https://typesense.org/docs/30.2/api/search.html

- The boolean tag search guide builds `filter_by` programmatically from user-picked tags, ORs values of one field and ANDs the fields, and escapes each value before interpolating it.
  https://typesense.org/docs/guide/boolean-tag-search.html

## Arrays and nested objects

- `genres:=[Rock, Pop]` is Rock OR Pop. `genres:=Rock && genres:=Acoustic` is Rock AND Acoustic.
- A numeric array filter such as `department_prices:<80` matches when ANY element matches.
- Nested object: `customer.name:=John Doe`.
- Array of objects: `ingredients.name:=cheese && ingredients.concentration:<50` can match on DIFFERENT items. For the same item use `ingredients.{name:=cheese && concentration:<50}`.
- Sub-fields of an `object[]` are arrays, so declare them `string[]`, `int32[]` even when single-valued, with `enable_nested_fields: true`.

https://typesense.org/docs/guide/tips-for-filtering.html and https://typesense.org/docs/30.2/api/search.html

## Empty values

`filter_by` rejects an empty value. Index a placeholder such as `~~` for empty values and add its symbol (`~`) to the field's `symbols_to_index`. Then `description:=~~` finds empty values and `description:!~~` the others.

```json
{"name": "description", "type": "string", "symbols_to_index": ["~"]}
```
https://typesense.org/docs/guide/tips-for-filtering.html

## Filters through joins

- `$categories(name:=Electronics && name:!=Disposable) && price:<100`. The `reference` must exist in the schema first. Nested joins work at any depth.
- Curated and pinned hits ignore `filter_by` unless `filter_curated_hits: true`.

https://typesense.org/docs/guide/tips-for-filtering.html and https://typesense.org/docs/30.2/api/search.html

## Geosearch

- Coordinates are `[latitude, longitude]`. GeoJSON order `[longitude, latitude]` is wrong here.
- Radius filter in `km` or `mi`: `location:(48.906, 2.343, 5.1 km)`.
- Nearest first: `sort_by: 'location(48.853, 2.344):asc'`. Hits carry `geo_distance_meters`.
- Polygon filter: `location:(48.8662, 2.3255, 48.8581, 2.3209, 48.8561, 2.3448, 48.8641, 2.3469)`.
- A `geopolygon` field stores an area as a flat list in clockwise or counter-clockwise order. A wrong order is an error. Point-in-polygon: `filter_by=area:(0.5, 0.5)`.
- `exclude_radius`: `location(48.853, 2.344, exclude_radius: 2mi):asc, popularity:desc` ties everything inside the radius so the next field decides.
- `precision`: `location(48.853, 2.344, precision: 2mi):asc, popularity:desc` buckets distances.
- Mix geo and other conditions in a sort: `_eval(has_relevant_experience:true && location:(48.853,2.344,5.1 km)):desc`.

https://typesense.org/docs/30.2/api/geosearch.html and https://typesense.org/docs/30.2/api/search.html

```js
client.collections('places').documents().search({
  q: '*', query_by: 'title',
  filter_by: 'location:(48.906, 2.343, 5.1 km)',
  sort_by: 'location(48.853, 2.344):asc'
})
```
https://typesense.org/docs/30.2/api/geosearch.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Filtering array-of-object conditions separately | `field.{a && b}` | https://typesense.org/docs/guide/tips-for-filtering.html |
| Prefix filter unavailable | `name:Jo*` (Typesense Server v27.0 or later) | https://typesense.org/docs/guide/tips-for-filtering.html |
| Algolia `insideBoundingBox` | polygon filter with the 4 corners | https://typesense.org/docs/guide/migrating-from-algolia.html |
