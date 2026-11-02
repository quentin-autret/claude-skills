# Search UI and migration from Algolia

InstantSearch through the Typesense adapter, and the behavior differences, concept mapping and data steps for moving from Algolia to Typesense 30.2.

## Contents

- InstantSearch adapter
- Behavior differences from Algolia
- Concept mapping
- Data migration steps

## InstantSearch adapter

Default UI: Algolia InstantSearch (JS, React, Vue, Angular) with `typesense-instantsearch-adapter`, which translates its queries.

```js
import TypesenseInstantSearchAdapter from 'typesense-instantsearch-adapter'
const typesenseInstantsearchAdapter = new TypesenseInstantSearchAdapter({
  server: {
    apiKey: 'abcd', // search-only key
    nodes: [{ host: 'localhost', port: 8108, protocol: 'http' }]
  },
  additionalSearchParameters: { query_by: 'title,authors' }
})
const search = instantsearch({ searchClient: typesenseInstantsearchAdapter.searchClient, indexName: 'books' })
```
https://typesense.org/docs/guide/search-ui-components.html

- `query_by` is required in `additionalSearchParameters`, which pass straight to the search endpoint.
- `indexName` is the collection name.
- `apiKey` is a search-only or scoped search key, never the admin key.
- React, Vue and Angular only swap the `searchClient` for the adapter's.
- With `create-instantsearch-app`, uncheck every "Attributes to display", "Dynamic Widgets" included. The docs mark this step as important.
- Some widgets need small changes, listed at https://github.com/typesense/typesense-instantsearch-adapter#widget-specific-instructions.
- Client-side analytics: the InstantSearch `analytics` widget and `insights` middleware.

https://typesense.org/docs/guide/search-ui-components.html, https://typesense.org/docs/guide/migrating-from-algolia.html and https://typesense.org/docs/guide/search-analytics.html

## Behavior differences from Algolia

- The APIs are not wire-compatible.
- Typesense type-checks documents against a schema, one type per field. Algolia stores any JSON. Bridge with auto schema detection and `dirty_values: coerce_or_reject` on import.
- Writes are synchronous: success means searchable, no task polling. Under heavy ingest, `503 Lagging or Not Ready` asks for a retry after a pause.
- Missing compared with Algolia: server-side A/B testing (use one collection per bucket on the client), dynamic synonym suggestions, built-in event tracking, personalization and recommendations (build them with ML and vector search).
- Most relevance settings are search-time parameters, not index settings. One collection supports many hard sort orders without replicas.

https://typesense.org/docs/guide/migrating-from-algolia.html and https://typesense.org/docs/30.2/api/documents.html#dealing-with-dirty-data

## Concept mapping

| Algolia | Typesense |
| --- | --- |
| index, record | collection, document |
| `objectID` (the page's jq step matches the key case-insensitively, written `objectId`) | `id` (string) |
| secured API keys | scoped search keys |
| query rules | curation sets, converter `npx algolia-query-rules-to-typesense@latest in.json out.json` |
| query suggestions | `popular_queries` analytics rule |
| merchandising | curation sets or `pinned_hits` and `hidden_hits` |
| optional filters, filter scoring | `_eval(...)` in `sort_by` |
| virtual replicas | `sort_by` on one collection |
| `multipleQueries` | `multi_search` |
| `searchableAttributes` | `query_by` |
| `attributesForFaceting` | `facet: true` plus `facet_by`, filter fields need no facet |
| `attributesToRetrieve`, `unretrievableAttributes` | scoped search key embedding `include_fields` or `exclude_fields` |
| `distinct` | `group_by` + `group_limit` |
| `separatorsToIndex` | `symbols_to_index` |
| `removeWordsIfNoResults` | `drop_tokens_threshold` |
| `disablePrefixOnAttributes` | `prefix=false,false,true` |
| `disableTypoToleranceOnAttributes` | per-field `num_typos`, `0` on the fields to protect |
| `customRanking` | `sort_by=_text_match(buckets: 10):desc,f1:desc,f2:desc` |
| `saveObjects` | import with `action=upsert` |
| `partialUpdateObjects` with create | import with `action=emplace` |
| `browseObjects` | export endpoint |
| `searchForFacetValues` | `facet_query` |
| `insideBoundingBox` | polygon filter with the 4 corners |
| `aroundPrecision` | `geo_precision`, `exclude_radius` |

https://typesense.org/docs/guide/migrating-from-algolia.html

The migration page writes the per-field typo mapping as `num_typos=false,false,true`. `num_typos` takes 0, 1 or 2.
https://typesense.org/docs/guide/migrating-from-algolia.html and https://typesense.org/docs/30.2/api/search.html

## Data migration steps

1. Export: `algolia objects browse YOUR_INDEX_NAME > documents-raw.jsonl`.
2. Transform: copy `objectId` (any casing) into `id` with the page's jq one-liner, convert ISO 8601 dates to Unix epoch integers for date sorting.
3. Create the collection first, with `facet: true` on facet fields. A 404 on import means it does not exist. Draft a schema with `npx typesense-collection-schema-generator@latest <input.json> <output-schema.json>`.
4. Import the JSONL in chunks with `action=upsert` and check every result line.
5. Import the converted rules as curation sets.

https://typesense.org/docs/guide/migrating-from-algolia.html

Long term, make the backend send the same JSON to Typesense instead of relying on the one-time export.
https://typesense.org/docs/guide/migrating-from-algolia.html
