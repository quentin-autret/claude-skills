# Tokenization, locale and stemming

How Typesense splits text, why SKUs, emails and middle-of-word matches fail by default, language handling and stemming in Typesense 30.2.

## Contents

- Default tokenization
- Recipes for tricky values
- Infix search
- Long texts, HTML and empty values
- Locale
- Stemming
- Old patterns

## Default tokenization

- Special characters are removed at indexing and at search time. `K83913.39F29.59444AT` is indexed as `K8391339F2959444AT`.
- Search is prefix-based: a term matches only the start of a word, so `39F29` inside that SKU finds nothing.
- Default fix: `token_separators`, which splits on extra characters. Field-level values win over collection-level values.

```json
{ "name": "products",
  "fields": [ {"name": "title", "type": "string"}, {"name": "part_number", "type": "string"} ],
  "token_separators": ["."] }
```
https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html

- `symbols_to_index` keeps a character verbatim instead of dropping it, for example `+` so `c++` is searchable.
- Field-level form: `{"name": "title", "type": "string", "token_separators": ["-", "."], "symbols_to_index": ["_"]}`.

https://typesense.org/docs/30.2/api/collections.html

- Prefix matching applies only to the last word of the query. `prefix=false` turns it off.
  https://typesense.org/docs/guide/faqs.html

## Recipes for tricky values

| Data | Schema setting | Source |
| --- | --- | --- |
| SKU, part number | `"token_separators": ["."]` | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |
| Phone number | `"token_separators": ["(", ")", "-"]` plus extra formats (digits only, without country code, without area code) in a `string[]` field | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |
| Email | `"token_separators": ["+", "-", "@", "."]`, so `contact+docs-example@typesense.org` indexes as contact, docs, example, typesense, org | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |
| URL, file path | `"token_separators": [":", "/", "."]` | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |

For words that contain special characters, prefer `token_separators` over infix.
https://typesense.org/docs/guide/faqs.html

## Infix search

- Infix matches inside a token. It needs `"infix": true` in the field schema AND the `infix` search parameter (`off` by default, `always` or `fallback` per `query_by` field).
- It uses only the first word of the query, costs extra memory and CPU, and suits short fields (emails, phone numbers, identifiers). Benchmark first.
- Alternative without infix: store suffix variants of the identifier in a `string[]` field, combined with `token_separators`.

```json
{"name": "part_number", "type": "string", "infix": true}
```
https://typesense.org/docs/30.2/api/search.html, https://typesense.org/docs/guide/faqs.html and https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html

## Long texts, HTML and empty values

- Long texts (articles, transcripts): split into paragraph documents. Otherwise common keywords match most articles and relevance drops.
- HTML: index a plain-text field without tags and query it. Keep the raw HTML out of the schema so it is only returned.
- Searching for null or empty values has no native support. Index a boolean such as `is_tags_null` and filter `is_tags_null:true`.

https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html

## Locale

- `locale` is set per field, so one document can mix languages: `{ "name": "title", "type": "string", "locale": "vi" }`.
- Default `en`. English fields drop diacritics on European characters. Non-English locales keep them (ICU tokenization), so an exact accented match ranks first and typo tolerance still finds the unaccented form.
- Any two-letter ISO 639-1 code works, passed to ICU (the page lists ar, zh, nl, en, fr, de, hi, id, it, ja, ko, ms, pl, pt, ru, el, es, th, tr, vi).
- Always set `locale` on non-English fields. Put the language in field names for multilingual content: `title_en`, `title_vi`.
- Japanese is `ja`. One table of the collections page shows `jp`, which is wrong.

https://typesense.org/docs/guide/locale.html and https://typesense.org/docs/30.2/api/collections.html

- Own tokenizer (CJK, compound words): send `pre_segmented_query: true` and space-separated tokens. Tokenization must be identical at index and query time. Typesense then splits on spaces only.
  https://typesense.org/docs/guide/locale.html and https://typesense.org/docs/30.2/api/search.html

## Stemming

- Default: `"stem": true` on the field. The Snowball stemmer uses the field `locale` for its language. The stored value is unchanged.
- Rules-based stemming targets common nouns and can hurt brand names, proper nouns, company names and places.
- Custom dictionary for domain words and irregular forms: a JSONL file of `{"word": "people", "root": "person"}` lines, imported then referenced with `stem_dictionary` on the field.

```bash
curl "http://localhost:8108/stemming/dictionaries/import?id=irregular-plurals" -X POST \
  -H "Content-Type: application/json" -H "X-TYPESENSE-API-KEY: ${TYPESENSE_API_KEY}" \
  --data-binary @dictionary.jsonl
```
```json
{"name": "title", "type": "string", "stem_dictionary": "irregular-plurals"}
```
https://typesense.org/docs/30.2/api/stemming.html

- `stem_dictionary` alone implies `stem: true`. With both, the dictionary wins.
- A ready-made English plurals dictionary: https://dl.typesense.org/data/stemming/plurals_en_v1.jsonl.
- Read back with `GET /stemming/dictionaries/<id>`, list with `GET /stemming/dictionaries`.
- The stemming page also says "Porter stemmer" once. The rest of the docs say Snowball.

https://typesense.org/docs/30.2/api/stemming.html

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Collection-level `token_separators` and `symbols_to_index` only | field-level values, which win | https://typesense.org/docs/30.2/api/collections.html |
| Pre-splitting every identifier by hand | `token_separators`, then infix for true middle-of-token matches | https://typesense.org/docs/guide/tips-for-searching-common-types-of-data.html |
