# API keys and access control

Bootstrap, admin and search-only keys, actions and collection scopes, scoped search keys for multi-tenancy, rotation and scraping protection in Typesense 30.2.

## Contents

- Sending the key
- Bootstrap and admin keys
- Creating keys
- Scope pitfalls
- Scoped search keys
- Multi-tenant patterns
- Keys in the frontend
- Rotation and scraping
- Old patterns

## Sending the key

Every endpoint except `GET /health` needs a key, in the `X-TYPESENSE-API-KEY` header or the `x-typesense-api-key` query parameter.
https://typesense.org/docs/guide/data-access-control.html

## Bootstrap and admin keys

- The bootstrap key is the `--api-key` given at server start. It can do everything.
- Never use it in applications. Use it once to create an admin key through `/keys`, so that key can be rotated or revoked.
- An admin key is `actions: ['*']` with `collections: ['*']`. The docs advise against such wide keys.

https://typesense.org/docs/guide/data-access-control.html and https://typesense.org/docs/30.2/api/api-keys.html

## Creating keys

```js
client.keys().create({
  description: 'Search-only companies key.',
  actions: ['documents:search'],
  collections: ['companies']
})
```
https://typesense.org/docs/30.2/api/api-keys.html

- `actions`, `collections` and `description` are required. `value` sets a custom key string. `expires_at` is a Unix timestamp. `autodelete: true` purges expired keys hourly.
- The full key value is returned ONLY in the create response. Retrieve and list return `value_prefix` (4 characters) and metadata.
- Keys are addressed by numeric `id`: `client.keys(1).retrieve()`, `client.keys(1).delete()`. There is no update, so replace a key by creating a new one and deleting the old one.
- Actions are `resource:verb`, for example `documents:search|get|create|upsert|update|delete|import|export|*`, `collections:create|delete|get|list|*`, `aliases:*`, `synonym_sets:*`, `synonym_sets/items:*`, `curation_sets:*`, `curation_sets/items:*`, `stopwords:*`, `keys:*`, `analytics/rules:*`, `analytics/events:create`, `presets:*`, `conversations/models:*`, `nl_search_models:*`, `operations/snapshot:create`, `metrics.json:list`.

https://typesense.org/docs/30.2/api/api-keys.html

## Scope pitfalls

- `collections` takes RegEx: `coll.*` matches every collection that has "coll" in its name. Write a prefix explicitly, like `org_.*`, and check what a pattern matches.
- `collections` restricts collection endpoints only. A key limited to some collections but holding `keys:*`, `synonym_sets:*` or `stopwords:*` still acts globally. Restrict `actions` too.
- `curation_sets:*` does not include `curation_sets/items:*`, and `synonym_sets:*` does not include `synonym_sets/items:*`. Grant both when both are needed.
- A search key on a collection with a `reference` field can read the referenced collection's fields through a joined query, though it cannot search that collection directly.

https://typesense.org/docs/30.2/api/api-keys.html and https://typesense.org/docs/30.2/api/joins.html

## Scoped search keys

A scoped search key is derived from a parent search key with embedded search parameters (HMAC). Typesense enforces them and the client cannot override them.

```js
// parent key: actions ['documents:search'] and nothing else
const scopedKey = client.keys().generateScopedSearchKey(parentSearchKey, {
  filter_by: 'company_id:124',
  filter_curated_hits: true,
  expires_at: 1906054106
})
```
https://typesense.org/docs/30.2/api/api-keys.html

- Generation is local, without a server call. Generate it on the backend, one per user.
- The parent key MUST hold only `documents:search`.
- The scoped `expires_at` must be earlier than the parent's.
- Curated hits and `pinned_hits` ignore the embedded `filter_by` by default. Embed `filter_curated_hits: true` as well.
- Hide the access list from responses by embedding `exclude_fields: 'accessible_to_user_ids'`.
- Revoking the parent key invalidates every scoped key derived from it.
- Any search parameter can be embedded: `include_fields`, `exclude_fields`, `limit_hits`, `limit_multi_searches`, `cache_ttl`.

https://typesense.org/docs/30.2/api/api-keys.html, https://typesense.org/docs/guide/data-access-control.html and https://typesense.org/docs/30.2/api/search.html

## Multi-tenant patterns

- Document level: one collection, an access field per document, a scoped search key per user: `filter_by: 'accessible_to_user_ids:=1'`.
- Roles: documents carry `accessible_to_organization_id` and an array `accessible_to_roles`. One parent search key per organization. At login, `filter_by: 'accessible_to_organization_id:=1 && accessible_to_roles:=[sales,marketing]'`. When a user leaves, delete that organization's parent key and issue a new one.
- Field level: a scoped search key per role embedding `exclude_fields: 'billing_address'`.
- Endpoint level: `actions`. Collection level: `collections`.

https://typesense.org/docs/30.2/api/api-keys.html and https://typesense.org/docs/guide/data-access-control.html

## Keys in the frontend

- Browsers and mobile apps may call Typesense directly. A search-only key is safe to expose when it grants exactly the data the search UI shows.
- With scoped search keys, NEVER expose the parent key: it allows unscoped searches over every document.
- Never expose the admin or bootstrap key.
- Mobile apps fetch hostnames and the current key from the backend at load or login, never hard-coded.
- The browser JS client is `SearchClient`, smaller and search-only.

https://typesense.org/docs/guide/data-access-control.html and https://typesense.org/docs/30.2/api/authentication.html

## Rotation and scraping

- Set `expires_at` with short windows and rotate. Add `autodelete: true` so expired keys do not pile up.
- Against scraping, embed `limit_hits` in a scoped search key, and `limit_multi_searches` for `multi_search`.
- Against DDOS, put each node hostname behind a proxying DNS such as Cloudflare (proxy on, SSL Full) and configure clients with those hostnames.

https://typesense.org/docs/guide/data-access-control.html and https://typesense.org/docs/30.2/api/api-keys.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `synonyms:*` actions | `synonym_sets:*` + `synonym_sets/items:*` | https://typesense.org/docs/30.2/api/synonyms.html |
| `overrides:*` actions | `curation_sets:*` + `curation_sets/items:*` | https://typesense.org/docs/30.2/api/curation.html |
| Bootstrap key in the application | admin key created through `/keys` | https://typesense.org/docs/guide/data-access-control.html |
| Algolia secured API keys | scoped search keys | https://typesense.org/docs/guide/migrating-from-algolia.html |
