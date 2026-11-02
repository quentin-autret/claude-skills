# TypeScript and tooling

## Contents

- AppConfig augmentation
- Type-safe message arguments
- Testing with Jest and Vitest
- Storybook
- Core library use-intl
- Runtime requirements
- Linting and translation management
- Conventions for coding agents

## AppConfig augmentation

TypeScript works without setup. Augmentation is optional and adds autocompletion and strict checks. https://next-intl.dev/docs/workflows/typescript

```ts
// global.ts
import {routing} from '@/i18n/routing';
import {formats} from '@/i18n/request';
import messages from './messages/en.json';

declare module 'next-intl' {
  interface AppConfig {
    Locale: (typeof routing.locales)[number];
    Messages: typeof messages;
    Formats: typeof formats;
  }
}
```
https://next-intl.dev/docs/workflows/typescript

- The interface must be named exactly `AppConfig`. All 3 members are optional. https://next-intl.dev/docs/workflows/typescript
- `Locale` comes from `routing.locales`, or from a shared `as const` array without routing. It types every API that takes or returns a locale (`useLocale`, `Link`'s `locale` prop) and the exported `Locale` type. https://next-intl.dev/docs/workflows/typescript
- `Messages` can be defined freely. When the messages are available locally, deriving the type from the default locale's JSON is convenient. https://next-intl.dev/docs/workflows/typescript
- `Formats`: export `formats` from the request config with `satisfies Formats`. Format names of `format.dateTime`, `number`, `list` and `displayName` become typed. https://next-intl.dev/docs/workflows/typescript
- Types not applied: check the interface name, that `tsconfig.json` includes the declaration file, then restart the editor. https://next-intl.dev/docs/workflows/typescript
- Large projects: enable `Messages` augmentation only in CI, or split messages into monorepo packages. Run costly type-aware `@typescript-eslint` rules only in CI (`process.env.CI ? 'error' : 'off'`). https://next-intl.dev/docs/workflows/typescript
- Benchmark on 340 messages: about 2.20 s without augmentation, 2.82 s with typed keys, 2.85 s with typed arguments. https://next-intl.dev/docs/workflows/typescript

## Type-safe message arguments

TypeScript infers imported JSON as loose `string` values (TypeScript issue 32063), so argument checks need a generated declaration. https://next-intl.dev/docs/workflows/typescript

```json
// tsconfig.json
{"compilerOptions": {"allowArbitraryExtensions": true}}
```

```js
// next.config.mjs
const withNextIntl = createNextIntlPlugin({
  experimental: {createMessagesDeclaration: './messages/en.json'}
});
```
https://next-intl.dev/docs/workflows/typescript

- The `.d.json.ts` file is generated on `next dev`, `next build` or `next typegen`. Ignore `messages/*.d.json.ts` in Git. https://next-intl.dev/docs/workflows/typescript
- The option is experimental. https://next-intl.dev/docs/workflows/typescript

## Testing with Jest and Vitest

```tsx
render(
  <NextIntlClientProvider locale="en" messages={messages}>
    <UserProfile />
  </NextIntlClientProvider>
);
```
https://next-intl.dev/docs/environments/testing

- Write components as non-async so tests need no Server Component mocking. https://next-intl.dev/docs/environments/testing
- next-intl is ESM-only and imports `next/navigation` without `.js` to avoid a Next.js deoptimization. https://next-intl.dev/docs/environments/testing
- Vitest with `createNavigation`: inline it with `test.server.deps.inline: ['next-intl']`. https://next-intl.dev/docs/environments/testing
- Jest has no built-in ESM: through `next/jest` with `testEnvironment: 'jsdom'`, set `transformIgnorePatterns: ['node_modules/(?!next-intl)/']`. https://next-intl.dev/docs/environments/testing

## Storybook

```tsx
// .storybook/preview.tsx
import {NextIntlClientProvider} from 'next-intl';

const preview: Preview = {
  decorators: [
    (Story) => (
      <NextIntlClientProvider locale="en" messages={defaultMessages}>
        <Story />
      </NextIntlClientProvider>
    )
  ]
};
```
https://next-intl.dev/docs/workflows/storybook

- Async Server Components in Storybook are experimental and may need extra config. Non-async shared components render as Client Components and read the decorator's provider. https://next-intl.dev/docs/workflows/storybook
- Community alternative: the `storybook-next-intl` addon (global provider plus locale switcher). https://next-intl.dev/docs/workflows/storybook

## Core library use-intl

- `use-intl` is the Next.js-agnostic core, for plain React, React Native, Jest or Storybook. Its provider is `IntlProvider`. https://next-intl.dev/docs/environments/core-library
- It has no routing APIs, no awaitable server APIs and no `i18n/request.ts` integration. https://next-intl.dev/docs/environments/core-library
- Outside React: https://next-intl.dev/docs/environments/core-library

```ts
import {createTranslator, createFormatter} from 'use-intl/core';
const t = createTranslator({locale: 'en', messages});
t.markup('rich', {name: 'world', b: (chunks) => `<b>${chunks}</b>`});
const format = createFormatter({locale: 'en'});
```
https://next-intl.dev/docs/environments/core-library

## Runtime requirements

- `Intl.Locale` is required for basic use. Features add `DateTimeFormat`, `NumberFormat`, `PluralRules`, `RelativeTimeFormat`, `ListFormat` and `DisplayNames`. https://next-intl.dev/docs/environments/runtime-requirements
- The package targets the browsers Next.js supports and Node.js 13 or later. https://next-intl.dev/docs/environments/runtime-requirements
- The Cloudflare cdnjs polyfill service, loaded with `next/script` `beforeInteractive`, does not cover every locale. https://next-intl.dev/docs/environments/runtime-requirements

## Linting and translation management

- The docs recommend `@eloqnt/cli`, built for next-intl, for linting catalogs: missing translations (`missing-translation`) and inconsistent ICU arguments (`inconsistent-args`). It reads `.json` and `.po`, so it covers `useTranslations` and `useExtracted`. https://next-intl.dev/docs/workflows/messages

```bash
npm install @eloqnt/cli --save-dev
npx eloqnt lint
```

```ts
// .eloqnt/config.ts
import {defineConfig} from '@eloqnt/cli';
export default defineConfig({
  srcPath: './src',
  messages: {path: './messages', locales: 'infer', sourceLocale: 'en', format: 'json'}
});
```
https://next-intl.dev/docs/workflows/messages

- Any localization platform that handles JSON works. The docs recommend Crowdin (CLI, GitHub app, webhooks, OTA SDK). https://next-intl.dev/docs/workflows/localization-management

```yml
# crowdin.yml
files:
  - source: /messages/en.json
    translation: /messages/%locale%.json
```
https://next-intl.dev/docs/workflows/localization-management

## Conventions for coding agents

- `useExtracted` suits agents: inline messages, no key naming, catalogs synced automatically. https://next-intl.dev/docs/workflows/agents
- With `useTranslations`, the docs suggest these project conventions: every user-facing string goes through `useTranslations`, each message has an entry in `messages/en.json` (other locales belong to translators), keys are short and descriptive (`title`, `description`), ICU arguments replace concatenation. https://next-intl.dev/docs/workflows/agents
- Do not translate catalogs with a generic agent: it lacks context, drifts in terminology and guesses plural rules. Leave translation to translators or a dedicated tool. https://next-intl.dev/docs/workflows/agents
