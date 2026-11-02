# Messages

## Contents

- Translating in components
- Keys and namespaces
- ICU arguments, plurals and select
- Rich text, markup and raw
- Arrays of messages
- Missing messages and errors
- Locale type, time zone and now
- RTL
- Old patterns

## Translating in components

```tsx
const t = useTranslations('About');
<h1>{t('title')}</h1>
```
https://next-intl.dev/docs/usage/translations

- Use `useTranslations` in non-async components (server or client) and `await getTranslations('About')` from `next-intl/server` in async components. https://next-intl.dev/docs/getting-started/app-router
- Translate inside React components. The intended exceptions are Server Actions, metadata and Route Handlers, which use `getTranslations`. Do not build translation helpers outside components. https://next-intl.dev/docs/usage/translations
- Pass the lowest common namespace that contains everything the component needs, and reach deeper with dots: `t('form.submit')`. https://next-intl.dev/docs/usage/translations

## Keys and namespaces

- Keys are IDs, not human-readable sentences. An editor integration can show the readable labels. https://next-intl.dev/docs/usage/translations
- Keys and namespaces cannot contain `.`, which expresses nesting. Any other character is fine. https://next-intl.dev/docs/usage/translations
- Nested messages are optional but give more structure. Flat dotted keys migrate to nested ones with lodash `set`. https://next-intl.dev/docs/usage/translations
- `t.has('title')` tells whether the message exists for the current locale. https://next-intl.dev/docs/usage/translations

## ICU arguments, plurals and select

```json
{
  "followers": "You have {count, plural, =0 {no followers yet} =1 {one follower} other {# followers}}.",
  "birthday": "It's your {year, selectordinal, one {#st} two {#nd} few {#rd} other {#th}} birthday!",
  "status": "{gender, select, female {She is} male {He is} other {They are}} online."
}
```
https://next-intl.dev/docs/usage/translations

- Argument names allow letters, digits and `_`. Dashes are not supported. https://next-intl.dev/docs/usage/translations
- `select` values follow the same rule: pass `locale.replaceAll('-', '_')` to turn `en-GB` into `en_GB`. https://next-intl.dev/docs/usage/translations
- `select` requires an `other` case. https://next-intl.dev/docs/usage/translations
- Plurals: `=0` for a friendly zero case, plus the CLDR tags the language needs (`zero`, `one`, `two`, `few`, `many`, `other`). `#` is the formatted number. https://next-intl.dev/docs/usage/translations
- In `selectordinal`, an `=value` case always takes precedence. https://next-intl.dev/docs/usage/translations
- Escape literal braces with single quotes: `'{name}'`. https://next-intl.dev/docs/usage/translations
- Use ICU arguments rather than string concatenation. https://next-intl.dev/docs/workflows/agents

## Rich text, markup and raw

Default: `t.rich`, which returns React nodes.

```tsx
// "message": "Please refer to <guidelines>the guidelines</guidelines>."
t.rich('message', {guidelines: (chunks) => <a href="/guidelines">{chunks}</a>});
```
https://next-intl.dev/docs/usage/translations

- Messages are always parsed, so a self-closing tag needs a closing tag: `"Hello,<br></br>how are you?"`. Tags can nest. https://next-intl.dev/docs/usage/translations
- Attributes such as `href` belong at the call site, not in the message. If one must live in messages, give it its own message and read it with `t('partnerHref')`. Localized paths come from `pathnames` routing. https://next-intl.dev/docs/usage/translations
- Shared tags: a `RichText` component with a render prop, `<RichText>{(tags) => t.rich('description', tags)}</RichText>`. Merge values with `{...tags, username}`. https://next-intl.dev/docs/usage/translations
- Escape hatch for HTML strings: `t.markup`. Its handlers take and return strings, not React nodes. https://next-intl.dev/docs/usage/translations

```tsx
t.markup('markup', {important: (chunks) => `<b>${chunks}</b>`});
```
https://next-intl.dev/docs/usage/translations

- Last resort: `t.raw('content')` skips parsing and returns any JSON value (string, boolean, object, array). Sanitize before `dangerouslySetInnerHTML`, or you open an XSS hole. https://next-intl.dev/docs/usage/translations

## Arrays of messages

- Map known keys to an array inside the component. This keeps ICU features and static validation. https://next-intl.dev/docs/usage/translations

```tsx
const keys = ['one', 'two', 'three'] as const;
keys.map((key) => <li key={key}>{t(`items.${key}`)}</li>);
```
https://next-intl.dev/docs/usage/translations

- When the item count differs per locale, read the keys with `Object.keys(useMessages().Namespace)`. https://next-intl.dev/docs/usage/translations

## Missing messages and errors

- Default behavior: the error is logged to the console and `namespace.key` renders. https://next-intl.dev/docs/usage/configuration
- Expected gaps (partly translated locales): merge the default-locale messages under the current ones in `getRequestConfig`. https://next-intl.dev/docs/usage/configuration
- `getMessageFallback` is for apps that treat missing messages as errors. https://next-intl.dev/docs/usage/configuration
- In `onError`, log `MISSING_MESSAGE` only and report other codes to error tracking: they indicate bugs. https://next-intl.dev/docs/usage/configuration
- `onError` and `getMessageFallback` are functions, so Client Components do not inherit them from a server-rendered provider. Wrap them in your own client provider, nested inside the server one, which inherits the rest. https://next-intl.dev/docs/usage/configuration

```tsx
'use client';
import {NextIntlClientProvider} from 'next-intl';

export default function IntlErrorHandlingProvider({children}) {
  return (
    <NextIntlClientProvider
      onError={(error) => console.error(error)}
      getMessageFallback={({namespace, key}) => `${namespace}.${key}`}
    >
      {children}
    </NextIntlClientProvider>
  );
}
```
https://next-intl.dev/docs/usage/configuration

## Locale type, time zone and now

- Type locale parameters with `import {Locale} from 'next-intl'`. It is `string` until you augment `AppConfig`. https://next-intl.dev/docs/usage/configuration
- Set `timeZone` explicitly in the request config. The default is the server runtime's zone. The client inherits it only through a server-rendered `NextIntlClientProvider`. https://next-intl.dev/docs/usage/configuration
- `useNow` and `getNow` return the current time without any config. A global `now` mainly serves consistent tests. Passing `now` per call can help caching. https://next-intl.dev/docs/usage/configuration
- Global `formats` (dateTime, number, list, displayName) are usable in `useFormatter`. Number, date and time formats can also be referenced in messages: `{orderDate, date, short}`. https://next-intl.dev/docs/usage/configuration

## RTL

- Beyond translations: set `dir`, mirror the layout with CSS logical properties and mirror directional elements such as icons. https://next-intl.dev/docs/usage/translations

## Old patterns

| Old | Current |
| :-- | :-- |
| `t.raw` HTML injected unsanitized | `t.rich` for React, `t.markup` for strings (https://next-intl.dev/docs/usage/translations) |
| Flat dotted keys | Nested objects (https://next-intl.dev/docs/usage/translations) |
