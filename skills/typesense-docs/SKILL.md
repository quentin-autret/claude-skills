---
name: typesense-docs
description: Pitfalls, surprising defaults and recommended patterns of the Typesense search engine, distilled from the official docs. Use when designing a Typesense collection schema, writing search parameters (q, query_by, filter_by, sort_by, facet_by, group_by), tuning typo tolerance and ranking, building vector, semantic or hybrid search, conversational RAG or natural language search, creating API keys and scoped search keys for multi-tenancy, indexing and import (JSONL, upsert, emplace, dirty_values), syncing from a primary database, reindexing through aliases, managing synonyms (synonym sets), curation sets and stopwords, joins, analytics rules and query suggestions, InstantSearch, migrating from Algolia, or running, clustering, backing up and upgrading a Typesense server.
metadata:
  version: "30.2"
  source: https://typesense.org/docs/
  fetched: "2026-11-02"
---

# Typesense from the official docs

## Overview

This skill holds what the Typesense docs say that is easy to get wrong: what costs RAM, import results that hide failures behind HTTP 200, filter precedence, defaults that loosen results, scoped search key leaks, hybrid search weights and the v30 move to synonym sets and curation sets. Generic search and database advice is left out. Every claim carries the docs URL it comes from.

## Version covered

Typesense Server 30.2, with the JavaScript client for code samples. Server flags apply to self-hosted servers. Typesense Cloud manages them. Before reviewing code written for an older server, read "Old patterns" below.

## How to navigate

1. Read the quick start below. It covers the rules that matter in most tasks.
2. Open the one reference file whose domain matches the task, from the table at the end. Each file stands alone.
3. Every claim ends with its source URL. Reopen it when a detail decides the outcome.

## Quick start

### Schema and memory

- Every field in the schema is indexed in RAM. Display-only fields stay out of the schema (still stored and returned) or get `index: false`. Prefer a pre-defined schema over `.*` auto detection.
  https://typesense.org/docs/30.2/api/collections.html and https://typesense.org/docs/guide/running-in-production.html
- No date type: store Unix timestamps in `int64`. `geopoint` is `[lat, lng]`, never GeoJSON order.
  https://typesense.org/docs/30.2/api/collections.html and https://typesense.org/docs/30.2/api/geosearch.html
- `facet: true` is needed for `facet_by` and `group_by`. String sorting needs `sort: true`.
  https://typesense.org/docs/30.2/api/search.html
- An alter blocks all writes to the collection. Default for schema changes: a new versioned collection, reindex, then repoint the alias the app always uses.
  https://typesense.org/docs/30.2/api/collections.html
- Special characters are dropped and matching is prefix-only: SKUs, emails and URLs need `token_separators`.
  https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html

### Writing data

- Default: import (JSONL) with `action: 'upsert'`. `upsert` replaces the whole document, `emplace` or `update` take partial ones.
  https://typesense.org/docs/30.2/api/documents.html
- Import ALWAYS answers HTTP 200. Scan every result line for `success: false`, with `return_id=true`.
  https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/guide/faqs.html
- Set your own string `id`. Give imports a client timeout of at least 5 minutes (the sync guide says as high as 60 minutes for large batches), and never retry a write that is still running.
  https://typesense.org/docs/30.2/api/documents.html and https://typesense.org/docs/guide/syncing-data-into-typesense.html
- `503 Not Ready or Lagging` is write backpressure: batch writes, add CPU, retry with jitter.
  https://typesense.org/docs/guide/syncing-data-into-typesense.html

### Searching

- `filter_by` needs the colon: `price:>=100`. `:` matches words, `:=` the whole value. `&&` and `||` share one precedence, so add parentheses. Backticks escape values with commas or brackets.
  https://typesense.org/docs/guide/tips-for-filtering.html
- Same-object conditions in an `object[]`: `ingredients.{name:=cheese && concentration:<50}`.
  https://typesense.org/docs/guide/tips-for-filtering.html
- `drop_tokens_threshold` and `typo_tokens_threshold` default to 1, so a query with few hits silently drops words or adds typos. Require all words with `drop_tokens_threshold: 0`. Disable typos with `num_typos: 0` AND `typo_tokens_threshold: 0`.
  https://typesense.org/docs/guide/ranking-and-relevance.html
- `max_candidates` defaults to 4. Short queries miss records unless you raise it or set a popularity `default_sorting_field`.
  https://typesense.org/docs/30.2/api/search.html
- Blend relevance and popularity: `_text_match(buckets: 10):desc,popularity:desc`. Boost without filtering: `_eval(in_stock:true):desc`.
  https://typesense.org/docs/guide/ranking-and-relevance.html
- Max 250 hits per page. `group_by` returns `grouped_hits` and an approximate `found`.
  https://typesense.org/docs/30.2/api/search.html
- `union: true` multi-search ignores per-search pagination and needs identical sort field types.
  https://typesense.org/docs/30.2/api/federated-multi-search.html

### Vector and AI search

- Vector queries go through `multi_search`, always with `exclude_fields` on the vector field. Bound results with `k` or `distance_threshold`.
  https://typesense.org/docs/30.2/api/vector-search.html
- Hybrid: `query_by: 'title,embedding'`. Fusion is 0.7 keyword and 0.3 vector, and `alpha` in `vector_query` is the VECTOR weight. `query_by_weights` needs a placeholder `0` for the embedding field.
  https://typesense.org/docs/30.2/api/vector-search.html
- RAG: `conversation=true` with `exclude_fields: 'embedding'`, or raw floats fill the LLM context.
  https://typesense.org/docs/30.2/api/conversational-search-rag.html

### Keys and multi-tenancy

- Never use the bootstrap key in apps. Never expose admin keys, nor the parent of scoped search keys.
  https://typesense.org/docs/guide/data-access-control.html
- Scoped search key: parent with only `documents:search`, generated on the backend per user, embedding `filter_by` AND `filter_curated_hits: true`, or pinned and curated hits leak across tenants.
  https://typesense.org/docs/30.2/api/api-keys.html and https://typesense.org/docs/30.2/api/curation.html
- `collections` on a key does not restrict global endpoints (`keys`, `synonym_sets`, `stopwords`). A reference field exposes the referenced collection through joins.
  https://typesense.org/docs/30.2/api/api-keys.html

### Operations

- Production: 3 or 5 node HA, clients configured with every node (plus `nearestNode` behind a load balancer), RAM under 85%.
  https://typesense.org/docs/guide/high-availability.html and https://typesense.org/docs/guide/running-in-production.html
- Back up with `/operations/snapshot`, never by copying the live data dir. Upgrade followers first, the leader last.
  https://typesense.org/docs/guide/backups.html and https://typesense.org/docs/guide/updating-typesense.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `/collections/{c}/synonyms`, `synonyms:*` key actions | `/synonym_sets`, `synonym_sets:*` and `synonym_sets/items:*`, linked by collection `synonym_sets` | https://typesense.org/docs/30.2/api/synonyms.html |
| `/collections/{c}/overrides`, `overrides:*` | `/curation_sets`, `curation_sets:*` and `curation_sets/items:*` | https://typesense.org/docs/30.2/api/curation.html |
| `override_tags` | `curation_tags` | https://typesense.org/docs/30.2/api/curation.html |
| analytics rules with `params.source` and `params.destination` | top-level `collection`, `event_type`, flat `params` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
| one counter rule with weighted `source.events` | one rule per event type with `params.weight` | https://typesense.org/docs/30.2/api/analytics-query-suggestions.html |
| exact `found` with `group_by` | approximate, `group_max_candidates` for accuracy | https://typesense.org/docs/30.2/api/search.html |
| LLM call in the app to build filters | `nl_search_models` + `nl_query=true` | https://typesense.org/docs/guide/natural-language-search.html |
| flattened nested objects | `enable_nested_fields` | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |
| one collection with `document_type` for a mixed list | `multi_search` with `union: true` | https://typesense.org/docs/guide/faqs.html |
| old `highlights` array | `highlight` object, `enable_highlight_v1=false` | https://typesense.org/docs/30.2/api/search.html |

Keys cannot be updated: replace old keys by creating new ones. Self-hosted servers take a snapshot before upgrading so synonyms and overrides migrate.
https://typesense.org/docs/30.2/api/synonyms.html

## Reference files

| File | Read it for |
| --- | --- |
| [references/schema-and-collections.md](references/schema-and-collections.md) | field parameters and defaults, types, RAM cost, auto schema, nested objects, alter, aliases, clone, collection layout |
| [references/tokenization-locale-stemming.md](references/tokenization-locale-stemming.md) | `token_separators`, `symbols_to_index`, infix, SKU, email and URL recipes, locale and diacritics, stemming dictionaries |
| [references/documents-and-import.md](references/documents-and-import.md) | ids, import actions, reading import results, timeouts, `dirty_values`, update, delete and export by query |
| [references/syncing-and-reindexing.md](references/syncing-and-reindexing.md) | sync strategies from a database, high-volume writes, 503 and resource errors, alias reindex |
| [references/search-parameters.md](references/search-parameters.md) | `q`, `query_by`, typo tolerance, token dropping, prefix, `max_candidates`, highlighting, pagination, cache, presets |
| [references/filtering-and-geo.md](references/filtering-and-geo.md) | `filter_by` syntax, precedence, escaping, arrays, nested objects, empty values, geo filters and sorts |
| [references/ranking-and-sorting.md](references/ranking-and-sorting.md) | default ranking, `sort_by`, `text_match_type`, buckets, `_eval`, decay, missing values, pinning |
| [references/faceting-grouping-multi-search.md](references/faceting-grouping-multi-search.md) | facets and strategies, `group_by`, federated and union multi-search, boolean tag search |
| [references/synonyms-curation-stopwords.md](references/synonyms-curation-stopwords.md) | synonym sets, curation sets, tags, dynamic rules, curation with scoped search keys, stopwords |
| [references/vector-and-hybrid-search.md](references/vector-and-hybrid-search.md) | vector fields, auto-embedding models, `vector_query`, hybrid fusion and `alpha`, HNSW, memory |
| [references/conversational-nl-image-voice.md](references/conversational-nl-image-voice.md) | RAG conversation models, natural language search, image and voice queries, recommendations |
| [references/api-keys-and-access-control.md](references/api-keys-and-access-control.md) | key types, actions, scope pitfalls, scoped search keys, multi-tenant patterns, rotation |
| [references/clients-and-errors.md](references/clients-and-errors.md) | client libraries, node and load balancer config, JS tuning, cache, error codes, slow searches |
| [references/server-configuration.md](references/server-configuration.md) | server flags and defaults, CORS, TLS, logging, resource limits, analytics flags, install notes |
| [references/cluster-ha-backups-upgrades.md](references/cluster-ha-backups-upgrades.md) | HA clusters, nodes file, recovery, operations API, monitoring, backups, upgrades, sizing, CI |
| [references/analytics-and-query-suggestions.md](references/analytics-and-query-suggestions.md) | analytics rules and events, popular and no-hits queries, counters, log rules, suggestions |
| [references/joins-and-personalization.md](references/joins-and-personalization.md) | reference fields, joined includes, filters and sorts, left and anti joins, async references, personalization |
| [references/ui-and-algolia-migration.md](references/ui-and-algolia-migration.md) | InstantSearch adapter, Algolia differences, concept mapping, migration steps |

Sources and fetch details are listed in [SOURCES.md](SOURCES.md).
