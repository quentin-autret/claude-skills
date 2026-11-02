# Locators

Choosing, filtering and combining locators in Playwright Test 1.64.1.

## Contents

- Priority order
- Text matching rules
- Locators are lazy and strict
- Filtering and the relative `has` pitfall
- Combining locators
- Lists and iteration
- Shadow DOM and frames
- Low-priority selectors and their replacements

## Priority order

Default: `getByRole` with an accessible name. Then `getByLabel` for form fields, `getByPlaceholder` only for inputs without a label, `getByText` for non-interactive elements, `getByAltText`, `getByTitle`, and `getByTestId` when role or text is impossible or a test-id contract exists.
https://playwright.dev/docs/locators#quick-guide

```js
await page.getByLabel('User Name').fill('John');
await page.getByRole('button', { name: 'Sign in' }).click();
await page.getByRole('button', { name: /submit/i }).click();
```
https://playwright.dev/docs/locators#locate-by-role

- Use `getByText` for `div`, `span`, `p`. For `button`, `a`, `input` use a role locator.
  https://playwright.dev/docs/locators#locate-by-text
- Role locators give early feedback only. They do not replace accessibility audits.
  https://playwright.dev/docs/locators#locate-by-role
- The default test id attribute is `data-testid`. Change it with `use.testIdAttribute`.
  ```js
  export default defineConfig({ use: { testIdAttribute: 'data-pw' } });
  ```
  https://playwright.dev/docs/locators#set-a-custom-test-id-attribute
- `page.locator(css or xpath)` is the escape hatch, only when nothing user-facing works. Long CSS chains are called a bad practice. The `css=` / `xpath=` prefix is auto-detected.
  https://playwright.dev/docs/locators#locate-by-css-or-xpath

## Text matching rules

- Default is substring and case-insensitive. `{ exact: true }` makes it exact and case-sensitive.
- Whitespace is ALWAYS normalized, even with `exact: true`: runs of spaces collapse, line breaks become spaces, leading and trailing space is ignored.

```js
await expect(page.getByText('Welcome, John', { exact: true })).toBeVisible();
await expect(page.getByText(/welcome, [A-Za-z]+$/i)).toBeVisible();
```
https://playwright.dev/docs/locators#locate-by-text

## Locators are lazy and strict

- A locator is not bound to an element. Every action re-resolves it against the current DOM, so re-renders between calls are fine.
  https://playwright.dev/docs/locators#locating-elements
- Strictness: an action that targets one element throws if the locator matches more than one. `page.getByRole('button').click()` throws on a page with several buttons. Multi-element operations such as `count()` are fine.
  https://playwright.dev/docs/locators#strictness
- `first()`, `last()`, `nth()` opt out of strictness and are "not recommended": the page can change and the locator then points at another element. Use them only for identical elements where order is the sole difference.
  https://playwright.dev/docs/locators#get-by-nth-item

## Filtering and the relative `has` pitfall

```js
await page.getByRole('listitem')
  .filter({ hasText: 'Product 2' })
  .getByRole('button', { name: 'Add to cart' })
  .click();
await expect(page.getByRole('listitem').filter({ hasNotText: 'Out of stock' })).toHaveCount(5);
```
https://playwright.dev/docs/locators#filter-by-text

- `hasText` is substring, case-insensitive, may match in a descendant, and accepts a regex.
- `has` / `hasNot` take another locator. That inner locator is queried FROM the outer match, not from the document root.
  `filter({ has: page.getByRole('list').getByText('Product 2') })` is wrong when the `<ul>` sits outside the matched `<li>`.
  https://playwright.dev/docs/locators#filter-by-childdescendant
- Filters chain: `.filter({ hasText: 'Mary' }).filter({ has: page.getByRole('button', { name: 'Say goodbye' }) })`.
  https://playwright.dev/docs/locators#chaining-filters
- Parent element: use `filter({ has: child })` rather than `xpath=..`.
  ```js
  const child = page.getByText('Hello');
  const parent = page.getByRole('listitem').filter({ has: child });
  ```
  https://playwright.dev/docs/other-locators#parent-element-locator

## Combining locators

- `inner.within(outer)` searches the inner locator inside the outer one. Relative pickers (`nth`, `first`) resolve inside each parent separately.
  ```js
  const saveButton = page.getByRole('button', { name: 'Save' });
  await saveButton.within(page.getByTestId('settings-dialog')).click();
  const thirdColumn = page.getByRole('cell').nth(2).within(page.getByRole('row'));
  ```
  https://playwright.dev/docs/locators#matching-inside-a-locator and https://playwright.dev/docs/release-notes#version-164
- `a.and(b)` must match both. `a.or(b)` matches either, and throws a strict mode violation when both are present, so add `.first()`.
  ```js
  const newEmail = page.getByRole('button', { name: 'New' });
  const dialog = page.getByText('Confirm security settings');
  await expect(newEmail.or(dialog).first()).toBeVisible();
  ```
  https://playwright.dev/docs/locators#matching-one-of-the-two-alternative-locators
- `locator.visible()` keeps visible matches only and replaces the `:visible` pseudo-class. `filter({ visible: false })` keeps invisible ones. The docs add that a more reliable identification usually beats a visibility check.
  ```js
  await page.locator('button').visible().click();
  ```
  https://playwright.dev/docs/locators#matching-only-visible-elements
- Save a narrowed locator and assert it is unique with `toHaveCount(1)`.
  https://playwright.dev/docs/locators#matching-inside-a-locator
- `locator.describe('Subscribe button')` names a locator in traces and reports.
  https://playwright.dev/docs/release-notes#version-153

## Lists and iteration

- Count: `await expect(page.getByRole('listitem')).toHaveCount(3);`
  https://playwright.dev/docs/locators#count-items-in-a-list
- All texts in order: `await expect(page.getByRole('listitem')).toHaveText(['apple', 'banana', 'orange']);`
  https://playwright.dev/docs/locators#assert-all-text-in-a-list
- Iterating is rare: `for (const row of await page.getByRole('listitem').all())`. `evaluateAll` runs in the page with any DOM API.
  ```js
  const texts = await rows.evaluateAll(list => list.map(element => element.textContent));
  ```
  https://playwright.dev/docs/locators#do-something-with-each-element-in-the-list

## Shadow DOM and frames

- All locators pierce OPEN shadow DOM by default. XPath does not pierce shadow roots. Closed shadow roots are not supported.
  https://playwright.dev/docs/locators#locate-in-shadow-dom
- Inside an iframe use `page.frameLocator(selector)`. Page-level actions run in the main frame.
  ```js
  const username = page.frameLocator('.frame-class').getByLabel('User Name');
  await username.fill('John');
  ```
  https://playwright.dev/docs/frames#introduction
- `page.frameLocator()` with no selector searches any frame of the subtree and errors when matches exist in several frames.
  https://playwright.dev/docs/release-notes#version-163
- `page.frame('name')` or `page.frame({ url: /domain/ })` returns a `Frame` with selector-string methods.
  https://playwright.dev/docs/frames#frame-objects

## Low-priority selectors and their replacements

Everything below is ranked under user-visible locators by the docs themselves.
https://playwright.dev/docs/other-locators

- `:has-text()` alone matches many elements, `<body>` included. Always prefix a tag: `article:has-text("Playwright")`.
  https://playwright.dev/docs/other-locators#css-matching-by-text
- `:text("Home")` is substring, case-insensitive. `:text-is("Home")` is exact and case-sensitive. `input type=button|submit` matches by `value`.
  https://playwright.dev/docs/other-locators#css-matching-by-text
- Index bases differ: `:nth-match(:text("Buy"), 3)` is ONE-based, the `nth=0` engine is ZERO-based (`nth=-1` is last).
  https://playwright.dev/docs/other-locators#css-pick-n-th-match-from-the-query-result and https://playwright.dev/docs/other-locators#n-th-element-locator
- Strings starting with `//` or `..` are XPath. A string that starts and ends with a quote, like `"Log in"`, becomes a legacy text locator.
  https://playwright.dev/docs/other-locators#xpath-locator and https://playwright.dev/docs/other-locators#legacy-text-locator
- Label retargeting: `click`, `fill`, `inputValue`, `selectText`, `setInputFiles`, `selectOption` on a `<label>` act on its control, but `toHaveText` asserts the label itself. Prefer `getByLabel`.
  https://playwright.dev/docs/other-locators#label-to-form-control-retargeting

| Old selector | Use instead | Source |
| --- | --- | --- |
| `:visible` pseudo-class | `locator.visible()` | https://playwright.dev/docs/other-locators#css-matching-only-visible-elements |
| `text=Log in`, `text="Log in"` | `getByText` | https://playwright.dev/docs/other-locators#legacy-text-locator |
| `id=`, `data-testid=`, `data-test=` shorthands | `getByTestId` | https://playwright.dev/docs/other-locators#id-data-testid-data-test-id-data-test-selectors |
| `:right-of()`, `:left-of()`, `:above()`, `:below()`, `:near()` (deprecated, may be removed) | user-visible locators | https://playwright.dev/docs/other-locators#css-matching-elements-based-on-layout |
| `css=a >> css=.b` chaining, `*` capture | locator chaining, `filter({ has })` | https://playwright.dev/docs/other-locators#chaining-selectors |
| `_react`, `_vue` selectors, `:light` suffix (removed) | standard locators | https://playwright.dev/docs/release-notes#version-158 |
