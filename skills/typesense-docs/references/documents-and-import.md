# Documents and import

Document ids, bulk import and its result handling, write actions, dirty data, update, delete and export in Typesense 30.2.

## Contents

- Document ids
- Single write versus import
- Import actions
- Reading the import result
- Batch size and timeouts
- Dirty data
- Update, delete, export
- Old patterns

## Document ids

- A string `id` field becomes the document identifier, otherwise Typesense generates one. `id` needs no schema declaration.
- The `id` must not contain spaces or any character that needs URL encoding.
- Always set `id` yourself on documents you will update. Otherwise you must save the generated id (`return_id=true` on import) in your database.
- Numeric database keys become strings: `id: row.id.toString()`.

https://typesense.org/docs/30.2/api/documents.html, https://typesense.org/docs/guide/syncing-data-into-typesense.html and https://typesense.org/docs/guide/node-prisma-search-api.html

## Single write versus import

- Default: import, `POST /collections/:collection/documents/import`. The single-document endpoint is only for a document written in response to a user action.
- Import takes JSONL, one object per line. Official clients accept an array and convert it. The output of export feeds import directly.
- One import request is at most 10GB. Split larger files.

```js
client.collections('companies').documents().import(documents, { action: 'upsert' })
client.collections('companies').documents().upsert(document) // single document
```
https://typesense.org/docs/30.2/api/documents.html

## Import actions

| `action` | Behavior |
| --- | --- |
| `create` (default) | creates, fails when the id exists |
| `upsert` | creates or replaces, needs the whole document |
| `update` | updates an existing document, fails when the id is missing, partial document allowed |
| `emplace` | creates or updates, whole or partial document |

https://typesense.org/docs/30.2/api/documents.html

- `upsert` replaces the document: fields missing from the payload are lost. Send partial documents with `update` or `emplace`.
  https://typesense.org/docs/30.2/api/documents.html

## Reading the import result

- Import always answers HTTP 200, whatever happened to each document. The HTTP client does not throw.
- Each response line, in request order, is `{"success": true}` or `{"success": false, "error": "...", "document": "..."}`. A failed document does not affect the others.
- Always scan the result for `success: false`. Missing records after a sync are usually schema validation errors hidden behind the 200.
- `return_id=true` adds the `id` to each line. `return_doc` returns the whole document.

```shell
curl -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" -X POST --data-binary @documents.jsonl \
  'http://localhost:8108/collections/companies/documents/import?return_id=true'
```
https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/guide/faqs.html

```js
const results = await client.collections('companies').documents().import(docs, { action: 'upsert', return_id: true })
const failed = results.filter(r => !r.success) // log r.id and r.error, then retry or fix
```
Pattern derived from https://typesense.org/docs/30.2/api/documents.html

- A 404 on import means the collection does not exist yet. Create it first.
  https://typesense.org/docs/guide/migrating-from-algolia.html

## Batch size and timeouts

- Server-side `batch_size` (default 40) makes Typesense ingest 40 documents, serve the search queue, then resume. Leave it. A larger value can hurt search and uses more transient memory.
- Batch on the client instead: several import calls with a controlled number of documents each.

https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/guide/syncing-data-into-typesense.html

- Writes are synchronous. Set the client timeout for imports to at least 5 minutes (`connectionTimeoutSeconds`), up to 60 minutes for large batches. A client that gives up early and retries the same write causes a thundering herd.
  https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/guide/syncing-data-into-typesense.html

## Dirty data

`dirty_values` works on create, update, upsert and import.

| Value | Behavior |
| --- | --- |
| `coerce_or_reject` | coerce to the field type, reject the document when that fails |
| `coerce_or_drop` | coerce, else drop the field and index the rest |
| `drop` | drop the mismatching field |
| `reject` | reject the document |

https://typesense.org/docs/30.2/api/documents.html

- The default depends on the schema: `coerce_or_reject` when the schema has a wildcard `.*` or any RegEx field name, `reject` otherwise.
- `string*` fields accept single values and arrays and convert everything to strings, handy for heterogeneous sources.

```js
client.collections('titles').documents().create({ title: 1984, points: 100 }, { dirty_values: 'coerce_or_reject' })
```
https://typesense.org/docs/30.2/api/documents.html

## Update, delete, export

- Partial update of one document: `documents('124').update({ num_employees: 5500 })` (PATCH). Several documents: import with `update`, `upsert` or `emplace`.
- Update by query returns `{"num_updated": N}`:
  ```js
  client.collections('companies').documents().update({ tag: 'large' }, { filter_by: 'num_employees:>1000' })
  ```
- Delete one: `documents('124').delete()`. A missing id is an error unless `ignore_not_found=true`.
- Delete by query: `documents().delete({ filter_by: 'num_employees:>100' })` returns `{"num_deleted": 24}`. `batch_size` speeds it up at the cost of other operations.
- Delete by ids: `filter_by: 'id:=[id1,id2,id3]'`. The documents page writes `id: [..]`, the sync guides use `id:=[..]`.
- Delete everything but keep the schema: `documents().delete({ truncate: true })`.
- Export: `documents().export()` returns JSONL and takes `filter_by`, `include_fields`, `exclude_fields`.
- Retrieve: `documents('124').retrieve()`, also with `include_fields` and `exclude_fields`.

https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/guide/syncing-data-into-typesense.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| One write call per record | import with `action: 'upsert'` | https://typesense.org/docs/guide/syncing-data-into-typesense.html |
| Delete with a match-all filter (`popularity:>0`), still documented as a tip | `truncate: true`, the dedicated "Delete all documents" operation | https://typesense.org/docs/30.2/api/documents.html |
| Raising server-side `batch_size` | client-side batching | https://typesense.org/docs/30.2/api/documents.html |
