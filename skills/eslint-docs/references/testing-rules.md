# Testing rules with RuleTester

## Contents

- Setup and defaults
- Valid and invalid cases
- Fixes and suggestions
- Strict checks that throw
- Test framework hooks
- Old patterns

## Setup and defaults

- `RuleTester` ships with `eslint`. It runs under Mocha, Jest or plain `node` and needs at least 1 valid and 1 invalid case. https://eslint.org/docs/latest/extend/custom-rule-tutorial
- Default `languageOptions: { ecmaVersion: "latest", sourceType: "module" }`. Old ES5 tests: `new RuleTester({ languageOptions: { ecmaVersion: 5, sourceType: "script" } })`. https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `RuleTester.setDefaultConfig(config)` affects only instances created afterwards and throws `TypeError` for a non-object. `getDefaultConfig()` and `resetDefaultConfig()` exist. https://eslint.org/docs/latest/integrate/nodejs-api
- Any other case property goes to the linter as config (`languageOptions`, for example). A valid case with only `code` can be a string. https://eslint.org/docs/latest/integrate/nodejs-api

```js
const { RuleTester } = require("eslint");
const ruleTester = new RuleTester();
ruleTester.run("avoid-name", rule, {
  valid: ["bar", { code: "var foo = true", options: [{ allowFoo: true }] }],
  invalid: [{ code: "foo", errors: [{ messageId: "avoidName" }] }],
});
```

https://eslint.org/docs/latest/extend/custom-rules , https://eslint.org/docs/latest/integrate/nodejs-api

## Valid and invalid cases

- `options` holds the rule options without the severity. https://eslint.org/docs/latest/integrate/nodejs-api
- `errors` is required on invalid cases: a count (`errors: 1`) or an array. Each error needs `message` (string or RegExp) or `messageId`. https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `data` in an expected error builds the expected message. It does not assert the `data` passed to `context.report`. https://eslint.org/docs/latest/integrate/nodejs-api
- Since ESLint 10, valid cases may not have `errors` or `output`, and the `type` property on errors throws: remove it. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `assertionOptions` on `run()`: `requireMessage` (`true`, `"message"`, `"messageId"`), `requireLocation`, `requireData` (`true`, `"error"`, `"suggestion"`). To enforce them project-wide, subclass `RuleTester` and override `run()`. https://eslint.org/docs/latest/integrate/nodejs-api

## Fixes and suggestions

- `output` is the code after a single autofix pass. Omit it or set `null` when no fix applies. It must differ from `code`. https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Test each kind of fix in its own case: on overlapping fixes RuleTester applies only the first. https://eslint.org/docs/latest/integrate/nodejs-api
- If the rule produces suggestions, each error must list `suggestions`. Omitting it asserts there are none, a number only counts them. https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Each suggestion needs `output` and exactly 1 of `desc` or `messageId`. Add `data` to check the text. https://eslint.org/docs/latest/use/migrate-to-9.0.0

```js
{
  code: "var foo;",
  output: "var bar;",
  errors: [{ messageId: "shouldBeBar", line: 1, column: 5 }],
},
{
  code: "var foo;",
  errors: [{ suggestions: [{ messageId: "renameFoo", data: { newName: "bar" }, output: "var bar;" }] }],
}
```

https://eslint.org/docs/latest/integrate/nodejs-api

## Strict checks that throw

- Suggestion messages must be unique, and each suggestion must change the code and still parse. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Test cases must be unique. `filename` must be a string, `only` a boolean. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- No unsubstituted `{{ placeholder }}` may remain in a message. https://eslint.org/docs/latest/use/migrate-to-9.0.0

## Test framework hooks

- RuleTester uses `RuleTester.describe` and `RuleTester.it` first, then the global `describe` and `it`. Without them cases run in sequence and throw on failure. `only: true` uses `RuleTester.itOnly` or `it.only`. https://eslint.org/docs/latest/integrate/nodejs-api

```js
RuleTester.describe = function (text, method) {
  RuleTester.it.title = text;
  return method.call(this);
};
RuleTester.it = function (text, method) {
  test(RuleTester.it.title + ": " + text, method);
};
```

https://eslint.org/docs/latest/integrate/nodejs-api

## Old patterns

| Old | Current |
| :-- | :-- |
| `FlatRuleTester`, `eslint/use-at-your-own-risk` export | `RuleTester` from `eslint` |
| `parserOptions` in test cases | `languageOptions` |
| `errors: [{ type: "Identifier" }]` | Remove `type` |
| `output` equal to `code` for no fix | Omit `output`, or `null` |
| `errors` or `output` on valid cases | Remove them |

https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/use/migrate-to-10.0.0
