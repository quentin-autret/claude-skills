# Formatting

## Contents

- useFormatter
- Dates and times
- Relative time
- Numbers
- Lists and display names
- Formats inside messages
- Global formats

## useFormatter

- Format plain values (outside messages) with `const format = useFormatter()`: `dateTime`, `relativeTime`, `dateTimeRange`, `number`, `list`, `displayName`. https://next-intl.dev/docs/usage/dates-times
- Async components use `await getFormatter()` from `next-intl/server`. https://next-intl.dev/docs/environments/server-client-components
- Options are those of the `Intl.*` constructors (DateTimeFormat, NumberFormat, ListFormat, DisplayNames). https://next-intl.dev/docs/usage/numbers
- Each method accepts a global format name as second argument (third for `dateTimeRange`), with an extra options argument to override it. https://next-intl.dev/docs/usage/dates-times

## Dates and times

- Store dates as full ISO 8601 strings (`2020-11-20T10:36:01.516Z`) and parse them with `new Date()` before formatting. next-intl only formats. https://next-intl.dev/docs/usage/dates-times

```js
format.dateTime(dateTime, {year: 'numeric', month: 'short', day: 'numeric'});
format.dateTime(dateTime, 'short', {year: 'numeric'}); // global format + override
format.dateTimeRange(dateTimeA, dateTimeB, {year: 'numeric', month: 'short', day: 'numeric'});
```
https://next-intl.dev/docs/usage/dates-times

## Relative time

- Take "now" from `useNow()`, not `new Date()`. The value stays consistent across client re-renders, can refresh with `updateInterval` and can come from the global `now` config. https://next-intl.dev/docs/usage/dates-times

```js
const now = useNow({updateInterval: 1000 * 10});
format.relativeTime(dateTime, now);
format.relativeTime(dateTime, {now, unit: 'day'});
```
https://next-intl.dev/docs/usage/dates-times

- The unit is picked automatically and values are rounded: 126 minutes gives "2 hours ago". Force a unit with `{now, unit: 'day'}`. https://next-intl.dev/docs/usage/dates-times

## Numbers

```js
format.number(499.9, {style: 'currency', currency: 'USD'});
format.number(499.9, 'price', {currency: 'USD'}); // global format "price" + override
```
https://next-intl.dev/docs/usage/numbers

- Output varies by locale beyond separators: grouping (`hi-IN` gives "1,20,000") and currency position (`de-DE` "12 €", `de-AT` "€ 12"). https://next-intl.dev/docs/usage/numbers

## Lists and display names

```tsx
format.list(items, {type: 'conjunction'}); // or 'disjunction'

const items = users.map((user) => <a key={user.id} href={`/user/${user.id}`}>{user.name}</a>);
<p>{format.list(items)}</p>; // returns Iterable<ReactElement>
```
https://next-intl.dev/docs/usage/lists

```js
format.displayName('US', {type: 'region'});
format.displayName('US', 'region', {style: 'narrow'});
```
https://next-intl.dev/docs/usage/display-name

- Lists and display names have no inline message syntax: format them only with `useFormatter`. https://next-intl.dev/docs/usage/lists , https://next-intl.dev/docs/usage/display-name
- `displayName` requires `type`. Accepted values depend on the runtime, commonly `region`, `language`, `currency`, `script`. https://next-intl.dev/docs/usage/display-name
- To render an array of messages, map message keys to an array inside the component. https://next-intl.dev/docs/usage/translations

## Formats inside messages

```json
{
  "ordered": "Ordered on {orderDate, date, medium}",
  "orderedCustom": "Ordered on {orderDate, date, ::yyyyMMMd}",
  "percentage": "Displayed as a percentage: {value, number, percent}",
  "custom": "At most 2 fraction digits: {value, number, ::.##}"
}
```
https://next-intl.dev/docs/usage/dates-times , https://next-intl.dev/docs/usage/numbers

- Built-in date formats: `full`, `long`, `medium`, `short`. Built-in number formats: `percent` and `currency`. https://next-intl.dev/docs/usage/dates-times , https://next-intl.dev/docs/usage/numbers
- A skeleton needs the leading `::` to indicate that a skeleton is used. https://next-intl.dev/docs/usage/dates-times
- Date skeleton symbols: `G` era, `y` year, `M` month (`M` to `MMMMM`), `d` day, `E` weekday, `h` 1-12, `K` 0-11, `H` 0-23, `k` 1-24, `m` minute and `s` second (2 digits when combined, `mmss`), `z` time zone. `yyyyMMMd` gives "Jul 9, 2024". https://next-intl.dev/docs/usage/dates-times
- Named formats for one call go in the third argument of `t`: https://next-intl.dev/docs/usage/numbers

```js
t('price', {price: 32000.99}, {number: {currency: {style: 'currency', currency: 'EUR'}}});
t('ordered', {orderDate}, {dateTime: {short: {day: 'numeric', month: 'short', year: 'numeric'}}});
```
https://next-intl.dev/docs/usage/numbers , https://next-intl.dev/docs/usage/dates-times

## Global formats

- Default: define shared formats once in the request config (`formats: {dateTime, number, list, displayName}`) and refer to them by name, in `useFormatter` and in messages (`{latitude, number, precise}`). https://next-intl.dev/docs/usage/configuration
- Format names are loose `string` until you add `Formats` to `AppConfig` (export `formats` with `satisfies Formats`). https://next-intl.dev/docs/workflows/typescript
- Translators benefit from an editor that understands ICU date and number syntax. https://next-intl.dev/docs/usage/dates-times
