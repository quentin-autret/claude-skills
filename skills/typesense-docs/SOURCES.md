# Sources

- Product: Typesense Server (self-hosted and Typesense Cloud), JavaScript client for code samples
- Version covered: 30.2 (latest at fetch time, versioned API pages under `/docs/30.2/`)
- Fetch date: 2026-11-02
- Entry page: https://typesense.org/docs/
- Raw sources used to list pages: https://typesense.org/docs/llms.txt and the `.md` variant of each page. API pages were read in their `.javascript.md` variant when it existed.
- Pages whose `.javascript.md` variant returned the site HTML shell were read from the base `.md` variant instead: stemming, stopwords, joins, conversational search, natural language search, image search, voice query.
- Pages that failed to load: none.
- Excluded on purpose: marketing overview pages, live demos and reference implementations, framework search-bar tutorials, platform integrations (Laravel, Magento, WordPress, Firebase, Supabase, DynamoDB, MongoDB, Django, Gin, Spring Boot, Deno), Typesense Cloud dashboard pages and the Cloud Management API.

## Pages read

### Schema and collections

- https://typesense.org/docs/30.2/api/ (release notes, read as https://typesense.org/docs/30.2/api/README.md)
- https://typesense.org/docs/30.2/api/collections.html
- https://typesense.org/docs/30.2/api/collection-alias.html
- https://typesense.org/docs/guide/organizing-collections.html
- https://typesense.org/docs/guide/locale.html
- https://typesense.org/docs/30.2/api/stemming.html
- https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html

### Documents and sync

- https://typesense.org/docs/30.2/api/documents.html
- https://typesense.org/docs/guide/syncing-data-into-typesense.html
- https://typesense.org/docs/guide/node-prisma-search-api.html
- https://typesense.org/docs/guide/postgres-typesense-migration-fts.html
- https://typesense.org/docs/guide/faqs.html

### Search and relevance

- https://typesense.org/docs/30.2/api/search.html
- https://typesense.org/docs/guide/ranking-and-relevance.html
- https://typesense.org/docs/guide/tips-for-filtering.html
- https://typesense.org/docs/30.2/api/federated-multi-search.html
- https://typesense.org/docs/30.2/api/synonyms.html
- https://typesense.org/docs/30.2/api/stopwords.html
- https://typesense.org/docs/30.2/api/curation.html
- https://typesense.org/docs/guide/boolean-tag-search.html
- https://typesense.org/docs/30.2/api/geosearch.html

### Vector and AI search

- https://typesense.org/docs/30.2/api/vector-search.html
- https://typesense.org/docs/guide/semantic-search.html
- https://typesense.org/docs/30.2/api/conversational-search-rag.html
- https://typesense.org/docs/30.2/api/natural-language-search.html
- https://typesense.org/docs/guide/natural-language-search.html
- https://typesense.org/docs/30.2/api/image-search.html
- https://typesense.org/docs/30.2/api/voice-search-query.html
- https://typesense.org/docs/guide/ai-agents-typesense.html
- https://typesense.org/docs/guide/recommendations.html

### Security, keys and clients

- https://typesense.org/docs/30.2/api/api-keys.html
- https://typesense.org/docs/30.2/api/authentication.html
- https://typesense.org/docs/guide/data-access-control.html
- https://typesense.org/docs/30.2/api/api-errors.html
- https://typesense.org/docs/30.2/api/api-clients.html
- https://typesense.org/docs/guide/installing-a-client.html
- https://typesense.org/docs/guide/typesense-js-client-tuning.html

### Operations

- https://typesense.org/docs/30.2/api/server-configuration.html
- https://typesense.org/docs/30.2/api/cluster-operations.html
- https://typesense.org/docs/guide/running-in-production.html
- https://typesense.org/docs/guide/high-availability.html
- https://typesense.org/docs/guide/system-requirements.html
- https://typesense.org/docs/guide/backups.html
- https://typesense.org/docs/guide/updating-typesense.html
- https://typesense.org/docs/guide/install-typesense.html
- https://typesense.org/docs/guide/testcontainers.html
- https://typesense.org/docs/guide/github-actions.html

### Analytics, joins and UI

- https://typesense.org/docs/30.2/api/analytics-query-suggestions.html
- https://typesense.org/docs/guide/query-suggestions.html
- https://typesense.org/docs/guide/search-analytics.html
- https://typesense.org/docs/30.2/api/joins.html
- https://typesense.org/docs/guide/personalized-search-join.html
- https://typesense.org/docs/guide/personalization.html
- https://typesense.org/docs/guide/search-ui-components.html
- https://typesense.org/docs/guide/migrating-from-algolia.html

## Inconsistencies found in the docs

- Token truncation parameter: `truncate` in the v30 release notes, `truncate_len` on the collections page.
- Japanese locale: `jp` in one table of the collections page, `ja` everywhere else.
- Stemmer: "Porter" once on the stemming page, Snowball elsewhere.
- Delete by ids: `id: [..]` on the documents page, `id:=[..]` in the sync guides.
- Remote embedding timeout: the vector search page gives different defaults in prose and in its tables.
- The OpenAI-compatible embedding sample uses an `azure/` model name, against the prose rule.
- The stopwords page multi-search sample contains syntax errors.
- The Algolia migration page maps per-field typo settings to `num_typos=false,false,true`, while `num_typos` takes 0, 1 or 2.
- Some counter rule samples on the analytics page send events whose `name` differs from the rule name.
