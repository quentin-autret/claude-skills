# Vector, semantic and hybrid search

Embedding fields, auto-embedding models, `vector_query`, hybrid rank fusion, distance thresholds, HNSW tuning and memory sizing in Typesense 30.2.

## Contents

- Vector fields
- Auto-embedding
- Embedding models
- Querying vectors
- Semantic and hybrid search
- Sorting with vectors
- HNSW and brute force
- Memory and CPU
- Old patterns

## Vector fields

- Own embeddings: a `float[]` field with `num_dim` equal to the model dimension. Docs examples use 4 dimensions for readability, real use needs at least 256.
- `vec_dist` defaults to `cosine`, or `ip`.

```js
{ name: 'embedding', type: 'float[]', num_dim: 384 }
```
https://typesense.org/docs/30.2/api/vector-search.html and https://typesense.org/docs/30.2/api/collections.html

- Precomputed embeddings can be imported with `action=emplace` and only `id` plus the vector, other fields added later.
  https://typesense.org/docs/guide/recommendations.html

## Auto-embedding

- Default: let Typesense embed. `embed.from` lists `string` or `string[]` fields (concatenated with spaces), `embed.model_config.model_name` picks the model. `q` is embedded with the same model at search time.
- Embeddings are regenerated only when an `embed.from` field changes.

```js
{ name: 'embedding', type: 'float[]',
  embed: { from: ['product_name', 'categories'], model_config: { model_name: 'ts/e5-small' } } }
```
https://typesense.org/docs/30.2/api/vector-search.html

- E5-style models need prefixes: `indexing_prefix: "passage:"` and `query_prefix: "query:"` in `model_config`.
- Changing a remote model API key: `PATCH /collections/<name>` with the FULL field definition (`name`, `embed.from` and every `model_config` parameter).

```bash
curl "http://localhost:8108/collections/products" -X PATCH -H "Content-Type: application/json" \
  -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" \
  -d '{"fields":[{"name":"embedding","embed":{"from":["product_name"],"model_config":{"model_name":"openai/text-embedding-ada-002","api_key":"new_api_key"}}}]}'
```
https://typesense.org/docs/30.2/api/vector-search.html

## Embedding models

| `model_name` prefix | Needs | Source |
| --- | --- | --- |
| `ts/` built-in ONNX (`ts/all-MiniLM-L12-v2`, `ts/e5-small`, `ts/gte-small`, multilingual `ts/paraphrase-multilingual-mpnet-base-v2`) | nothing, downloaded after collection creation | https://typesense.org/docs/guide/semantic-search.html |
| `openai/` | `api_key` | https://typesense.org/docs/30.2/api/vector-search.html |
| `azure/` | `api_key`, `url` | https://typesense.org/docs/30.2/api/vector-search.html |
| OpenAI-compatible server | name starting with `openai`, custom `url`, a `POST /v1/embeddings` endpoint | https://typesense.org/docs/30.2/api/vector-search.html |
| `google/embedding-gecko-001` | `api_key`, the only model of that provider | https://typesense.org/docs/30.2/api/vector-search.html |
| `gcp/gemini-embedding-001` | `project_id`, `service_account{client_email, private_key}` | https://typesense.org/docs/30.2/api/vector-search.html |
| custom ONNX | `<data_dir>/models/<name>/` with `model.onnx`, `config.json` (`model_type` `bert` or `xlm_roberta`, `vocab_file_name`) | https://typesense.org/docs/30.2/api/vector-search.html |

- The OpenAI-compatible sample shows an `azure/` name, contradicting the prose. Follow the prose.
- Vertex AI prefers service account auth, with no refresh token to manage.
- Remote calls retry on failure. Tune with `remote_embedding_timeout_ms` and `remote_embedding_num_tries` at search time, `remote_embedding_batch_size` (default 200) at import. Defaults: `remote_embedding_num_tries` 2; `remote_embedding_timeout_ms` 30s in the search table and 60000 at import, while the prose says "about a minute", so set it explicitly.

https://typesense.org/docs/30.2/api/vector-search.html

## Querying vectors

- Send vector queries through `multi_search` (POST body), because vectors are large.
- ALWAYS `exclude_fields` the vector field, or every hit returns the raw floats.
- `vector_query: 'embedding:([0.96, 0.94, 0.39, 0.30], k:100)'`. Parameters: `k`, `id`, `alpha`, `distance_threshold`, `queries`, `query_weights`, `flat_search_cutoff`, `ef`.
- Every document matches a vector query. Bound the set with `k` and/or `distance_threshold`, then page inside it with `page` and `per_page`. With both `per_page` and `k`, the larger wins.
- Hits carry `vector_distance`, cosine from 0 (identical) to 2 (opposite), sorted ascending.
- Similar to an existing document: `embedding:([], id: foobar)` with `q: '*'`. The source document is not returned.
- `[]` on an auto-embedding field embeds `q` and still applies `k` and `distance_threshold`.
- Weighted history: `embedding:([], queries:[smart phone, apple ipad], query_weights:[0.9, 0.1])`.

```js
client.multiSearch.perform({ searches: [{
  collection: 'products', q: 'device to type things on', query_by: 'embedding',
  vector_query: 'embedding:([], k: 200)', exclude_fields: 'embedding', per_page: 10, page: 1
}] }, {})
```
https://typesense.org/docs/30.2/api/vector-search.html

## Semantic and hybrid search

- Semantic: `query_by` the auto-embedding field. The docs examples add `prefix: false`.
- Hybrid: `query_by: 'product_name,embedding'`. Rank fusion: `0.7 * keyword_rank + 0.3 * semantic_rank`. `alpha` in `vector_query` is the VECTOR weight: `alpha: 0.8` gives 0.8 to vectors.
- `_text_match` in `sort_by` then means the fusion score. Hits carry `hybrid_search_info.rank_fusion_score`.
- `query_by_weights` ignores the embedding field but must have the same length as `query_by`. Put `0` for the embedding field.
- Hybrid with an external query vector: pass it in `vector_query` and leave the vector field out of `query_by`.
- Multi-word hybrid queries: `drop_tokens_threshold: 0` avoids redundant keyword passes.
- By default a keyword-only hit has no vector distance and a vector-only hit no text score. `rerank_hybrid_matches: true` computes both for every hit, at extra cost.
- `distance_threshold` excludes far documents in vector search. Inside `sort_by` they get the max distance and stay.

```json
{"q":"chair","query_by":"product_name","vector_query":"embedding:([0.2, 0.4, 0.1])","sort_by":"_text_match:desc"}
```
https://typesense.org/docs/30.2/api/vector-search.html and https://typesense.org/docs/guide/semantic-search.html

## Sorting with vectors

- `sort_by: 'popularity_score:desc,_vector_distance:asc'`.
- Re-rank keyword results by similarity: `_text_match:desc,_vector_query(embedding:([])):asc` with a text-only `query_by`. User-level personalization uses this with a user-behavior embedding.
- Mix similarity with a score: `_vector_distance(bucket_size: 10), popularity:desc`.

https://typesense.org/docs/30.2/api/vector-search.html and https://typesense.org/docs/guide/personalization.html

## HNSW and brute force

- Default: approximate HNSW. `flat_search_cutoff: 20` scans exactly when the filter leaves fewer than 20 documents.
- Index time `hnsw_params`: `ef_construction` (default 200), `M` (default 16). Search time `ef` in `vector_query` (default 10).

```json
{"name":"vec","type":"float[]","num_dim":768,"hnsw_params":{"ef_construction":100,"M":8}}
```
https://typesense.org/docs/30.2/api/vector-search.html

## Memory and CPU

- Vector RAM: `7 bytes * dimensions * records` (384 dims, 100K records: 268.8 MB).
- Built-in models need an extra 2 GB to 6 GB of RAM. Remote embedders need none.
- Built-in models are CPU-heavy: a few thousand records can take tens of minutes. A GPU speeds up embedding only, never the nearest-neighbor search.

https://typesense.org/docs/guide/system-requirements.html, https://typesense.org/docs/guide/semantic-search.html and https://typesense.org/docs/30.2/api/vector-search.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Embedding every query in the app for semantic search | auto-embedding field in `query_by` | https://typesense.org/docs/30.2/api/vector-search.html |
| Vertex AI OAuth tokens with refresh | `gcp/gemini-embedding-001` with a service account | https://typesense.org/docs/30.2/api/vector-search.html |
| `text-embedding-ada-002` in samples | treat as a placeholder, use a current model | https://typesense.org/docs/30.2/api/vector-search.html |
