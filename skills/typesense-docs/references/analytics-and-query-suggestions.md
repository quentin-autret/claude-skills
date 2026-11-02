# Analytics rules and query suggestions

Rule types, events, popular and no-hits queries, counter-based popularity, log rules and query suggestion layouts in Typesense 30.2.

## Contents

- Enabling analytics
- Rule model
- Popular queries
- Meta fields
- No-hits queries
- Counter rules
- Log rules
- Query suggestions
- Old patterns

## Enabling analytics

- Off by default on a self-hosted server. Needs `--enable-search-analytics=true` AND `--analytics-dir=/path`.
- `--analytics-flush-interval` (default 3600s, minimum 60) decides when events are aggregated into destination collections. Lower means fresher.
- Typesense Cloud enables analytics with a 300s flush interval.
- Exclude a search from aggregation, for example from test scripts: `enable_analytics: false`.

```bash
./typesense-server --data-dir=/path/to/data --api-key=abcd \
  --enable-search-analytics=true --analytics-dir=/path/to/analytics-data --analytics-flush-interval=60
```
https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

## Rule model

- `type`: `popular_queries`, `nohits_queries`, `counter` or `log`.
- `event_type`: `search`, `click`, `conversion` or `visit`.
- A rule names its source `collection`, its `event_type`, an optional `rule_tag` and flat `params`.

```js
await client.analytics.rules().create({
  name: 'homepage_popular_queries',
  type: 'popular_queries',
  collection: 'products',
  event_type: 'search',
  rule_tag: 'homepage',
  params: { destination_collection: 'product_queries', limit: 100, capture_search_requests: true }
})
client.analytics.rules().upsert(ruleName, ruleConfiguration)
client.analytics.rules().retrieve({ rule_tag: 'homepage' })
client.analytics.rules('homepage_popular_queries').delete()
```
https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

- `create` also takes an array of rules.
- Deleting a rule stops aggregation. Data already aggregated stays in the destination collection.

https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

## Popular queries

- The destination collection is a normal collection filled by Typesense. Fields `q` (string) and `count` (int32) are mandatory.

```js
client.collections().create({ name: 'product_queries', fields: [ { name: 'q', type: 'string' }, { name: 'count', type: 'int32' } ] })
```
- Only queries that returned results are tracked.
- A query counts only after a pause of at least 4 seconds: `f`, `fo`, `foo`, pause, registers "foo". Local tests must wait for the pause AND the flush interval.
- `expand_query: true` stores the expanded word ("sho" leading to "shoe" hits stores "shoe"). Default `false` stores the raw text.
- Users are told apart by `x-typesense-user-id` (query parameter) or `X-TYPESENSE-USER-ID` (header), else by client IP.
- `capture_search_requests: false` stops auto capture. Events are then sent by API with the rule name:

```js
client.analytics.events().create({
  name: 'product_queries_aggregation', event_type: 'search',
  data: { q: 'running shoes', user_id: '1234' }
})
```
https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

## Meta fields

Only `filter_by` and `analytics_tag` are supported. If either of the first 2 conditions is missing, the meta fields are not stored:
1. string fields `filter_by` and `analytics_tag` in the destination schema
2. `params.meta_fields: ["analytics_tag", "filter_by"]` in the rule

Searches then send `analytics_tag=...` and `filter_by=...` as query parameters (as in the page example).

https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

## No-hits queries

Same shape with `type: 'nohits_queries'` and a `q` plus `count` destination. It reveals content gaps and candidates for synonyms.
https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

## Counter rules

- The counter field in the source schema must be `optional: true`: `{"name": "popularity", "type": "int32", "optional": true}`.
- One rule per event type, each with its own `weight` (click 1, conversion 2), optionally sharing a `rule_tag`.

```js
client.analytics.rules().upsert('product_clicks', {
  name: 'product_clicks', type: 'counter', collection: 'products', event_type: 'click',
  params: { destination_collection: 'products', counter_field: 'popularity', weight: 1 }
})
client.analytics.events().create({
  name: 'product_clicks', event_type: 'click', data: { doc_id: '1024', user_id: '111112' }
})
```
https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

- The event `name` is the rule name and `event_type` must equal the rule's. Some page samples mismatch them. Do not copy those.
- `data.doc_id` must be the real document `id`.
- Increments land at each flush, not instantly.
- Rank with the counter: `sort_by: '_text_match(buckets: 10):desc,popularity:desc'`.

https://typesense.org/docs/30.2/api/analytics-query-suggestions.html and https://typesense.org/docs/guide/ranking-and-relevance.html#ranking-based-on-relevance-and-popularity

## Log rules

- `{ name, type: 'log', collection: 'products', event_type: 'click' }` stores raw events for model training and personalization.
- Reading events works ONLY for `log` rules: `client.analytics.events().retrieve({ user_id: '1234', name: '<rule name>', n: 10 })`. `user_id`, `name` and `n` (max 1000) are required.

https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

## Query suggestions

- Low traffic, or typed queries covering little of the dataset: query the main collection on every keystroke and show one field in the dropdown.
- Enough traffic: query the popular queries collection next to the main one in one `multi_search`.

```js
client.multiSearch.perform({ searches: [
  { collection: 'product_queries', q: 'shoe', query_by: 'q' },
  { collection: 'products', q: 'shoe', query_by: 'product_name' }
]}, {})
```
https://typesense.org/docs/guide/query-suggestions.html and https://typesense.org/docs/30.2/api/analytics-query-suggestions.html

- Server analytics only knows searches. Conversion and engagement questions belong in a web analytics tool fed from the client, with a search-field debounce of about 1s.
  https://typesense.org/docs/guide/search-analytics.html

## Old patterns

Rules were migrated automatically on upgrade to v30 and keep working, with `rule_tag` set to the old rule name. Reading and updating them needs the new API and client methods. Old event names still work.

| Old | Current | Source |
| --- | --- | --- |
| `params.source.collections: [...]` | top-level `collection` + `event_type` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
| `params.destination.collection` | `params.destination_collection` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
| `source.enable_auto_aggregation: false` + `source.events` | `params.capture_search_requests: false` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
| One counter rule with several weighted `source.events` | one rule per event type with `params.weight` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
| `destination.counter_field` | `params.counter_field` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
