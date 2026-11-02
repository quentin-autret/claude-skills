# Conversational search, natural language search, image and voice queries

Built-in RAG, LLM-generated search parameters, CLIP image search, Whisper voice queries and vector recommendations in Typesense 30.2.

## Contents

- Conversational search (RAG)
- Natural language search
- Image search
- Voice queries
- Recommendations
- Old patterns

## Conversational search (RAG)

Setup:
1. A history collection with this fixed schema, populated by Typesense: `conversation_id` string, `model_id` string, `timestamp` int32, `role` string (`index: false`), `message` string (`index: false`).
2. A conversation model: `POST /conversations/models`.

```bash
curl 'http://localhost:8108/conversations/models' -X POST -H 'Content-Type: application/json' \
  -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" \
  -d '{"id":"conv-model-1","model_name":"openai/gpt-3.5-turbo","history_collection":"conversation_store",
       "api_key":"OPENAI_API_KEY","system_prompt":"Answer only from the provided context.","max_bytes":16384}'
```
https://typesense.org/docs/30.2/api/conversational-search-rag.html

- Providers: `openai/`, `azure/` (+ `url`), `google/`, `cloudflare/@cf/` (+ `account_id`), `vllm/` (+ `vllm_url`).
- `max_bytes` caps the bytes sent to the LLM per call, so check the model context window. `ttl` (default 86400s) expires history.
- Responses mask `api_key`. Without `id`, the generated id is the one to query with.
- Manage with `GET`, `PUT` (full parameters) and `DELETE /conversations/models/{id}`.

https://typesense.org/docs/30.2/api/conversational-search-rag.html

Asking:
- `conversation=true`, `conversation_model_id`, `q`, and `query_by` an auto-embedding field.
- In `multi_search`, `q`, `conversation` and `conversation_model_id` are URL query parameters, not per-search body fields. Several searches feed ONE answer.
- ALWAYS `exclude_fields: 'embedding'` and every field irrelevant to the answer, or raw floats fill the LLM context.
- Display `conversation.answer`. `exclude_fields: 'conversation_history'` drops the history from the response.

```bash
curl 'http://localhost:8108/multi_search?q=can+you+suggest+an+action+series&conversation=true&conversation_model_id=conv-model-1' \
  -X POST -H "Content-Type: application/json" -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" \
  -d '{"searches":[{"collection":"tv_shows","query_by":"embedding","exclude_fields":"embedding"}]}'
```
https://typesense.org/docs/30.2/api/conversational-search-rag.html

- Follow-ups: pass the previous `conversation_id`. Typesense calls the LLM once to rewrite the question with history, searches, then answers: 2 LLM calls per follow-up. Only the latest messages that fit in `max_bytes` (minus the system prompt) are used.
- Streaming: `conversation_stream=true` returns Server-Sent Events ending with `data: [DONE]`. The JS client takes `streamConfig: { onChunk, onError, onComplete }`, in the second argument for `multiSearch.perform`.

```ts
client.collections('tv_shows').documents().search({
  q: 'can you suggest an action series', query_by: 'embedding',
  conversation: true, conversation_model_id: 'conv-model-1', conversation_stream: true,
  streamConfig: { onChunk: c => console.log(c.conversation.message), onError: console.error, onComplete: console.log }
})
```
https://typesense.org/docs/30.2/api/conversational-search-rag.html

## Natural language search

Default: the built-in feature. Typesense asks an LLM to turn a sentence into `q`, `filter_by` and `sort_by`.

```bash
curl -X POST http://localhost:8108/nl_search_models -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"id":"gemini-model","model_name":"google/gemini-2.5-flash","api_key":"YOUR_KEY","max_bytes":16000,"temperature":0.0}'
curl "http://localhost:8108/collections/products/documents/search" -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" -G \
  --data-urlencode "q=show me red shirts under 50" --data-urlencode "nl_query=true" \
  --data-urlencode "query_by=name,description,color,category" --data-urlencode "nl_model_id=gemini-model"
```
https://typesense.org/docs/30.2/api/natural-language-search.html

- Providers: `openai/`, `azure/` (+ `url`), `cloudflare/@cf/` (+ `account_id`), `vllm/` (+ `api_url`), `google/`, `gcp/` (service account or OAuth, `project_id`, `region`).
- An optional `system_prompt` is appended to the prompt Typesense builds from the schema. Use it for domain terms.
- An explicit `filter_by` is ANDed with the generated one.
- The response adds `parsed_nl_query` (`generated_params`, `augmented_params`, `parse_time_ms`). Hide it with `exclude_fields=parsed_nl_query`.
- On LLM failure, Typesense falls back to the sentence as `q` (only if no explicit `q` was given), keeps explicit parameters and adds an `error` field. Handle degraded results.
- `nl_query_debug=true` returns the raw LLM response.
- The first request builds the prompt from `facet_by` queries over the schema, which can take seconds on large datasets. It is cached for `nl_query_prompt_cache_ttl` (default 86400s, `0` disables).

https://typesense.org/docs/30.2/api/natural-language-search.html

## Image search

- Built-in CLIP model `ts/clip-vit-b-p32` on an `image` field (base64 BMP, JPG or PNG). `store: false` uses the image for the embedding only and never stores it.

```json
{"name":"images","fields":[
  {"name":"name","type":"string"},
  {"name":"image","type":"image","store":false},
  {"name":"embedding","type":"float[]","embed":{"from":["image"],"model_config":{"model_name":"ts/clip-vit-b-p32"}}}]}
```
- `"from": ["name","image"]` combines text and image.
- Text to image: `q: 'animals'`, `query_by: 'embedding'`. Similar to a stored image: `q: '*'`, `vector_query: 'embedding:([], id:123)'`. Similar to an uploaded image: `q: '*'`, `vector_query: 'embedding:([], image:BASE64)'`.

https://typesense.org/docs/30.2/api/image-search.html

## Voice queries

- Link a model at collection creation: `"voice_query_model": {"model_name": "ts/whisper/base.en"}`.
- Audio must be 16 kHz, 16-bit, mono WAV, base64: `ffmpeg -i <file> -ar 16000 -ac 1 -c:a pcm_s16le voice_query.wav`.
- Send `voice_query` through `multi_search`. The GET search endpoint's length limit cannot hold base64 audio.

```json
{"searches":[{"collection":"products","query_by":"name","voice_query":"<base64 audio>"}]}
```
https://typesense.org/docs/30.2/api/voice-search-query.html

## Recommendations

- Train item embeddings on sessions, store them in a `float[]` field whose `num_dim` equals the training dimension, then query with the current session embedding: `embedding:([...], k:10, distance_threshold: 1)` with `q: '*'`.
- Use ids or SKUs in training data. Filter already-seen items in the client.
- Find similar to an item: `embedding:([], id: <item id>)`.

https://typesense.org/docs/guide/recommendations.html and https://typesense.org/docs/30.2/api/vector-search.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| External LLM call (Genkit) producing `filter_by` and `sort_by` | `nl_search_models` + `nl_query=true` | https://typesense.org/docs/guide/natural-language-search.html |
| `npm i typesense@next` | `npm install typesense` | https://typesense.org/docs/guide/ai-agents-typesense.html |
