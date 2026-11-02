# Message extraction (experimental)

## Contents

- Enabling extraction
- Writing inline messages
- Static analysis limits
- File formats and codecs
- Precompiling messages
- Monorepos and libraries
- Tests

## Enabling extraction

`useExtracted` and the plugin's `experimental` options are experimental. https://next-intl.dev/docs/usage/extraction

```ts
// next.config.ts
const withNextIntl = createNextIntlPlugin({
  experimental: {
    extract: true,
    messages: {path: './messages', format: 'json', locales: 'infer', sourceLocale: 'en'},
    srcPath: './src'
  }
});
```
https://next-intl.dev/docs/usage/extraction

- `srcPath` is required when `extract` is on. Use `extract` together with `messages` and `srcPath`. https://next-intl.dev/docs/usage/plugin
- `extract.path` defaults to `messages.path`. Set it only when `messages.path` is an array. https://next-intl.dev/docs/usage/plugin
- `locales: 'infer'` detects locales from the files in `path`. An explicit array restricts them, `[]` disables extraction. https://next-intl.dev/docs/usage/plugin
- Extraction runs through a Turbopack or Webpack loader on `next dev` and `next build`, with no manual step. It adds empty entries for target locales and removes outdated ones. https://next-intl.dev/docs/usage/extraction

## Writing inline messages

```tsx
import {useExtracted} from 'next-intl';
import {getExtracted} from 'next-intl/server';

const t = useExtracted();       // non-async components
const t = await getExtracted(); // async components, metadata, Server Actions

t('Hello {name}!', {name});
t.rich('Please refer to the <link>guidelines</link>.', {
  link: (chunks) => <Link href="/guidelines">{chunks}</Link>
});
t({message: 'Right', description: 'Advance to the next slide'});
```
https://next-intl.dev/docs/usage/extraction

- `useExtracted` compiles into `useTranslations` with a generated key. https://next-intl.dev/docs/usage/extraction
- Give translators context with `description`. https://next-intl.dev/docs/usage/extraction
- Escape hatch, rarely needed: an explicit ID, `t({id: 'carousel.next', message: 'Right'})`, when one label appears in several places but needs different translations. https://next-intl.dev/docs/usage/extraction
- Avoid namespaces (`useExtracted('ns')`) by default: they make moving messages between components harder. They help for monorepo libraries (key collisions) and for splitting client messages. https://next-intl.dev/docs/usage/extraction
- Enum-like values: map each value to an inline message where `t` is in scope (an async function with `getExtracted`, or a hook that returns a getter). Alternative: one message with ICU `select`. https://next-intl.dev/docs/usage/extraction
- Zod: keep the schema locale-agnostic and turn its structured errors into messages where `t` is in scope. https://next-intl.dev/docs/usage/extraction
- `t.raw` is not intended to be used with `useExtracted` (the one ICU exception). https://next-intl.dev/docs/usage/extraction
- `createMessagesDeclaration` is not needed with `useExtracted`. https://next-intl.dev/docs/usage/plugin

## Static analysis limits

- `t` must receive a literal string. `t(key)` with a runtime key is not extracted. https://next-intl.dev/docs/usage/extraction
- `t` must be called in the function body that obtained it. Passing `t` to another function is not supported. https://next-intl.dev/docs/usage/extraction
- Re-exporting the hook (`export const x = useExtracted`) is not supported. Calling `useExtracted` inside a custom hook is. https://next-intl.dev/docs/usage/extraction
- Destructuring from `Promise.all` works: `const [t, post] = await Promise.all([getExtracted(), getPost()])`. https://next-intl.dev/docs/usage/extraction

## File formats and codecs

- Recommended with `useExtracted`: PO files (`format: 'po'`); the docs' own config example uses `'json'`, so PO is not a default. Keys are generated, and PO carries file references and descriptions for (AI) translators. JSON holds only key/value pairs. https://next-intl.dev/docs/usage/extraction , https://next-intl.dev/docs/usage/plugin
- In PO, the key sits in `msgctxt` and the source string in `msgid`. File references are path-only (no line numbers) to avoid noisy diffs. A custom codec adds line numbers. https://next-intl.dev/docs/usage/plugin

```ts
// CustomCodec.ts
import {defineCodec} from 'next-intl/extractor';
export default defineCodec(() => ({
  decode(content, context) { /* ... */ },
  encode(messages, context) { /* ... */ }
}));
// config: messages.format = {codec: './CustomCodec.ts', extension: '.json'}
```
https://next-intl.dev/docs/usage/plugin

- `codec` may also be a package name. A `.ts` codec needs Node.js 22.18+ (native TypeScript). On older Node, write it in JavaScript. https://next-intl.dev/docs/usage/plugin

## Precompiling messages

- `messages.precompile: true` gives smaller bundles and faster runtime formatting. The loader preprocesses `(await import('../../messages/en.json')).default`. https://next-intl.dev/docs/usage/plugin
- `t.raw` is not supported with `precompile`. https://next-intl.dev/docs/usage/plugin
- Messages fetched at runtime: precompile them yourself with `icu-minify/compile`, at the same `icu-minify` version that `next-intl` depends on. https://next-intl.dev/docs/usage/plugin

## Monorepos and libraries

- One deployed app spread over packages: add the other packages to `srcPath`. https://next-intl.dev/docs/usage/extraction
- A shared package used by several apps ships its messages: run `unstable_extractMessages` (from `next-intl/extractor`) once at build, list the package in the consumer's `transpilePackages`, and load the shipped catalogs: https://next-intl.dev/docs/usage/extraction

```ts
experimental: {
  extract: {path: './messages'},
  messages: {
    path: ['./messages', '../ui/messages', './node_modules/@acme/components/messages'],
    format: 'po', locales: 'infer', sourceLocale: 'en'
  },
  srcPath: './src'
}
// i18n/request.ts spreads (await import(`@acme/ui/messages/${locale}.po`)).default into messages
```
https://next-intl.dev/docs/usage/extraction

- Manual extraction also suits component libraries that have no dev server. https://next-intl.dev/docs/usage/extraction

## Tests

- Without compilation, the inline message renders directly and the provider needs no messages: `<NextIntlClientProvider locale="en" timeZone="UTC">`. https://next-intl.dev/docs/usage/extraction
