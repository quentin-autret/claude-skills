# Clients, timeouts and errors

Client libraries, node configuration with and without a load balancer, JS client tuning, error codes and slow query diagnosis in Typesense 30.2.

## Contents

- Client libraries
- Node configuration
- JS client tuning
- Server-side cache
- Error codes
- Diagnosing slow searches

## Client libraries

- Official: JavaScript, PHP, Python, Ruby. Community: TypeScript, Go, .Net, Java, Rust, Dart, Swift, Clojure, Elixir and others. Clients are thin wrappers over the REST API.
- JavaScript: `npm install typesense @babel/runtime`. Browser script: `dist/typesense.min.js`.
- Go import path: `github.com/typesense/typesense-go/v3/typesense`.
- PHP needs an HTTP client: `composer require php-http/curl-client typesense/typesense-php`.
- The install page shows placeholder versions for Dart and Swift. Use the latest release instead.
- Integrations: `typesense-instantsearch-adapter`, Autocomplete.js, DocSearch, Laravel Scout.
- TypeScript is listed as a community client on the install page only, not on the API clients page.

https://typesense.org/docs/30.2/api/api-clients.html and https://typesense.org/docs/guide/installing-a-client.html

## Node configuration

Single node or self-hosted cluster without a load balancer: list every node. The client balances requests and retries transient failures.

```js
const Typesense = require('typesense')
const client = new Typesense.Client({
  nodes: [
    { host: 'typesense-1.example.net', port: 443, protocol: 'https' },
    { host: 'typesense-2.example.net', port: 443, protocol: 'https' },
    { host: 'typesense-3.example.net', port: 443, protocol: 'https' }
  ],
  apiKey: '<API_KEY>',
  connectionTimeoutSeconds: 2
})
```
https://typesense.org/docs/guide/high-availability.html

With a load balancer (Typesense Cloud HA): `nearestNode` is the load-balanced host AND `nodes` keeps every node as fallback, because the balancer can hit a node in its 30s connection-draining state during rotation.

```js
const client = new Typesense.Client({
  nearestNode: { host: 'xxx.a1.typesense.net', port: 443, protocol: 'https' },
  nodes: [
    { host: 'xxx-1.a1.typesense.net', port: 443, protocol: 'https' },
    { host: 'xxx-2.a1.typesense.net', port: 443, protocol: 'https' },
    { host: 'xxx-3.a1.typesense.net', port: 443, protocol: 'https' }
  ],
  apiKey: '<API_KEY>',
  connectionTimeoutSeconds: 2
})
```
https://typesense.org/docs/guide/high-availability.html

- Self-hosted default: `localhost:8108` over http. Typesense Cloud: per-node hostnames, port 443, https.
  https://typesense.org/docs/30.2/api/authentication.html
- Timeout by workload: about 2s for browser search, at least 5 minutes for imports, a large timeout for schema alters (one alter at a time per cluster, and it can take a long time).
  https://typesense.org/docs/guide/typesense-js-client-tuning.html, https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/30.2/api/collections.html

## JS client tuning

Browser search fails fast: short timeouts, small retry budget.

```js
import { SearchClient as TypesenseSearchClient } from 'typesense'
const client = new TypesenseSearchClient({
  nodes: [{ host: 'xxx.a1.typesense.net', port: 443, protocol: 'https' }],
  apiKey: 'your-search-only-api-key',
  connectionTimeoutSeconds: 2,
  timeoutSeconds: 5,
  numRetries: 3,
  retryIntervalSeconds: 1,
  cacheSearchResultsForSeconds: 120
})
```
https://typesense.org/docs/guide/typesense-js-client-tuning.html

- `SearchClient` holds only the search classes and keeps the bundle small.
- `cacheSearchResultsForSeconds` caches per browser session. It helps repeated searches and filter toggles but never reduces load across users.
- `healthcheckIntervalSeconds` matters for multi-node setups, so the client leaves an unhealthy node.

https://typesense.org/docs/guide/typesense-js-client-tuning.html and https://typesense.org/docs/30.2/api/authentication.html

## Server-side cache

- `useServerSideSearchCache: true` sends `use_cache=true`. Only searches with `use_cache` are cached (LRU, `--cache-num-entries` default 1000).
- `cache_ttl` (default 60s) is set inside a scoped search key. For data refreshed every 12 hours, 30 to 60 minutes is reasonable when slight staleness is fine.
- Clear the cache with `POST /operations/cache/clear`.

https://typesense.org/docs/guide/typesense-js-client-tuning.html, https://typesense.org/docs/30.2/api/search.html#caching-parameters and https://typesense.org/docs/30.2/api/cluster-operations.html

## Error codes

| Code | Meaning | Typical cause |
| --- | --- | --- |
| 400 | Bad Request | malformed syntax (the filter `price>=100` needs a colon: `price:>=100`) |
| 401 | Unauthorized | wrong key, or a key without the action, such as a `synonyms:*` key on `/synonym_sets` |
| 404 | Not Found | missing collection, for example importing before creating it |
| 409 | Conflict | resource exists, such as renaming a model to an existing id |
| 422 | Unprocessable Entity | well-formed request that cannot be processed |
| 503 | Service Unavailable | "Lagging or Not Ready" under high write volume, retry later |

https://typesense.org/docs/30.2/api/api-errors.html, https://typesense.org/docs/guide/tips-for-filtering.html, https://typesense.org/docs/30.2/api/synonyms.html, https://typesense.org/docs/30.2/api/conversational-search-rag.html, https://typesense.org/docs/guide/migrating-from-algolia.html and https://typesense.org/docs/guide/syncing-data-into-typesense.html

- Import answers 200 even when documents fail. Read each result line.
  https://typesense.org/docs/30.2/api/documents.html

## Diagnosing slow searches

```ini
log-dir = /var/log/typesense
enable-search-logging = true
enable-access-logging = true
log-slow-requests-time-ms = 2000
```
https://typesense.org/docs/guide/typesense-js-client-tuning.html

- Common causes: heavy faceting, large result windows, complex filters, hybrid search with remote embeddings. Remote embedding timeouts are set per search request.
- Slow requests show in the main log with the prefix `SLOW REQUEST`. `log-slow-requests-time-ms` defaults to `-1` (disabled).

https://typesense.org/docs/guide/typesense-js-client-tuning.html, https://typesense.org/docs/30.2/api/vector-search.html#remote-embedding-api-parameters and https://typesense.org/docs/30.2/api/cluster-operations.html
