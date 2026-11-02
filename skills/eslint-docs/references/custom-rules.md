# Writing custom rules

## Contents

- Rule shape and `meta`
- Using a local rule
- The `context` object and `context.report`
- Fixes and suggestions
- Options, `meta.schema`, `meta.defaultOptions`
- Deprecating a rule
- Old patterns

## Rule shape and `meta`

- A rule is an object `{ meta, create(context) }`. `create` returns visitors keyed by node type or selector. A `:exit` suffix runs on the way up. https://eslint.org/docs/latest/extend/custom-rules
- `meta.type`: `"problem"` (bug or confusing code), `"suggestion"` (a better way), `"layout"` (whitespace, semicolons, parentheses). `--fix-type` filters on it. https://eslint.org/docs/latest/extend/custom-rules
- `meta.fixable: "code"` or `"whitespace"` is required for a rule that returns fixes, and `meta.hasSuggestions: true` for one that gives suggestions. ESLint throws otherwise. https://eslint.org/docs/latest/extend/custom-rules
- `meta.docs.url` gives editors a link on each violation. https://eslint.org/docs/latest/extend/custom-rules
- `meta.languages` restricts a rule (`["js/js"]`, `"plugin/*"`, `"*"`). ESLint throws when no entry matches the active language. Omitted means every language. https://eslint.org/docs/latest/extend/custom-rules
- Do not import or extend core rules: they are not public API. Copy the file into your project instead. https://eslint.org/docs/latest/extend/custom-rules
- Explore ASTs, scopes and code paths with https://explorer.eslint.org. https://eslint.org/docs/latest/extend/custom-rules
- Search for an existing plugin before writing a rule. https://eslint.org/docs/latest/extend/custom-rule-tutorial

```js
export default {
  meta: {
    type: "suggestion",
    docs: { description: "Disallow variables named foo" },
    fixable: "code",
    messages: { avoidName: "Avoid using variables named '{{ name }}'" },
    schema: [],
  },
  create(context) {
    return {
      Identifier(node) {
        if (node.name === "foo") {
          context.report({ node, messageId: "avoidName", data: { name: "foo" } });
        }
      },
    };
  },
};
```

https://eslint.org/docs/latest/extend/custom-rules

## Using a local rule

- Wrap the rule in an inline plugin. The rule id takes the namespace prefix, and the rule runs only at `"warn"` or `"error"`. https://eslint.org/docs/latest/extend/custom-rules

```js
import { defineConfig } from "eslint/config";
import myRule from "./rules/my-rule.js";
export default defineConfig([
  { plugins: { local: { rules: { "my-rule": myRule } } }, rules: { "local/my-rule": "error" } },
]);
```

https://eslint.org/docs/latest/extend/custom-rules

## The `context` object and `context.report`

- Properties: `id`, `filename`, `physicalFilename`, `cwd`, `options`, `sourceCode`, `settings`, `languageOptions` (`sourceType`, `ecmaVersion`, `parser`, `parserOptions`, `globals`). `report()` is the only method. https://eslint.org/docs/latest/extend/custom-rules
- `context.options` excludes the severity: a rule cannot read or change its severity. Handle missing options with defaults. https://eslint.org/docs/latest/extend/custom-rules
- Report with `messageId` and `meta.messages` rather than `message`: tests do not repeat the text. https://eslint.org/docs/latest/extend/custom-rules
- `report()` needs `node` or `loc`. If both are given, `loc` wins. `loc.line` is 1-based, `column` 0-based. https://eslint.org/docs/latest/extend/custom-rules
- `{{ name }}` placeholders read `data`. A suggestion does not see the error's `data`: give it its own. https://eslint.org/docs/latest/extend/custom-rules
- Reporting on `Program`: since ESLint 10 its range covers the whole file, comments included. Report on `node.body[0] ?? node` so `/* eslint-disable your-rule */` still works. https://eslint.org/docs/latest/use/migrate-to-10.0.0

## Fixes and suggestions

- Fixer methods: `insertTextAfter`, `insertTextAfterRange`, `insertTextBefore`, `insertTextBeforeRange`, `remove`, `removeRange`, `replaceText`, `replaceTextRange`. Ranges are `[start inclusive, end exclusive]`. The text argument must be a string (TypeError since ESLint 10). https://eslint.org/docs/latest/extend/custom-rules , https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `fix` returns one fix, an array or an iterable. Fixes from one `fix()` must not overlap. https://eslint.org/docs/latest/extend/custom-rules
- Fixes are not applied at once: a fix that conflicts with another is dropped for this pass. Passes repeat up to 10 times and every rule reruns. https://eslint.org/docs/latest/extend/custom-rules
- Fix guidelines: never change runtime behavior, keep fixes small, 1 fix per message, ignore the style of the output (another rule such as `quotes` fixes it). https://eslint.org/docs/latest/extend/custom-rules
- Extend the fix range on purpose to block other fixes in the surrounding code during the same pass. https://eslint.org/docs/latest/extend/custom-rules
- Offer a suggestion (`suggest: [{ messageId, data, fix }]`) instead of a fix when the change could alter behavior or several fixes are valid. A suggestion is applied alone, without later passes: no big refactors, no matching of user style. A suggestion whose `fix` returns `null` or an empty array/sequence is dropped. https://eslint.org/docs/latest/extend/custom-rules

```js
context.report({
  node,
  messageId: "avoidName",
  *fix(fixer) {
    yield fixer.replaceText(node, replacementText);
    // extend the fix range to node.parent
    yield fixer.insertTextBefore(node.parent, "");
    yield fixer.insertTextAfter(node.parent, "");
  },
});
```

https://eslint.org/docs/latest/extend/custom-rules

## Options, `meta.schema`, `meta.defaultOptions`

- A rule that receives options must declare `meta.schema` (JSON Schema Draft-04), otherwise ESLint throws as soon as an option is passed. A rule without options omits `schema` or sets `[]`. Avoid `schema: false`. https://eslint.org/docs/latest/extend/custom-rules
- The schema never sees the severity. `"off"` stops validation. https://eslint.org/docs/latest/extend/custom-rules
- Array form: 1 schema per positional option. Fewer options than schemas is valid, more is an error, `[]` rejects any option. https://eslint.org/docs/latest/extend/custom-rules
- Object form must describe an array (`type: "array"`). A top-level `type: "object"` makes the rule impossible to enable. Use the object form when the schema needs `$ref`. https://eslint.org/docs/latest/extend/custom-rules
- Draft-04 only: no `if`, no `$data`. Ajv runs with `useDefaults`, which may go in a future major. https://eslint.org/docs/latest/extend/custom-rules
- `meta.defaultOptions` (an array) merges user options on top: objects merge recursively, arrays and primitives replace, `undefined` falls back. Defaults are validated by the schema. https://eslint.org/docs/latest/extend/custom-rules

```js
meta: {
  defaultOptions: [{ alias: "basic" }],
  schema: [{ type: "object", properties: { alias: { type: "string" } }, additionalProperties: false }],
},
create(context) {
  const [{ alias }] = context.options;
}
```

https://eslint.org/docs/latest/extend/custom-rules

## Deprecating a rule

- `meta.deprecated` is `true` or a `DeprecatedInfo` object: `message`, `url`, `replacedBy` (array, empty means no replacement), `deprecatedSince` (semver without `v`), `availableUntil` (semver, or `null`: the rule will no longer change but stays available). https://eslint.org/docs/latest/extend/rule-deprecation
- `replacedBy` entries: `message`, `url`, `plugin` (`{ name, url }`, name `"eslint"` for core, omit for the same plugin), `rule` (omit when the name is unchanged or the plugin has 1 rule). https://eslint.org/docs/latest/extend/rule-deprecation

## Old patterns

| Old | Current |
| :-- | :-- |
| `module.exports = function (context) {}` | `{ meta, create(context) }` |
| Options without `meta.schema` | Declare `meta.schema` |
| `meta.replacedBy` | `meta.deprecated.replacedBy` |
| `context.getSourceCode()` | `context.sourceCode` |
| `context.getFilename()`, `getPhysicalFilename()`, `getCwd()` | `context.filename`, `physicalFilename`, `cwd` |
| `context.parserOptions` | `context.languageOptions.parserOptions` |
| `context.parserPath`, `sourceCode.getJSDocComment()` | No replacement |
| `context.getScope()` | `sourceCode.getScope(node)` |
| `context.getAncestors()` | `sourceCode.getAncestors(node)` |
| `context.markVariableAsUsed(name)` | `sourceCode.markVariableAsUsed(name, node)` |
| `context.getSource()`, `getSourceLines()` | `sourceCode.getText()`, `getLines()` |
| Token and comment methods on `context` | Same names on `sourceCode` |
| `context.parserServices` | `sourceCode.parserServices` |
| `getTokenOrCommentBefore(n, skip)` | `getTokenBefore(n, { includeComments: true, skip })` |
| `isSpaceBetweenTokens(a, b)` | `isSpaceBetween(a, b)` |
| `sourceCode.getComments()`, `CodePath#currentSegments` | Comment variants, segment events |

https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/use/migrate-to-10.0.0

- `eslint-transforms v9-rule-migration rule.js` rewrites the removed `context` methods, not the properties. Helper lint rules: `eslint-plugin/prefer-object-rule`, `eslint-plugin/require-meta-schema`. https://eslint.org/docs/latest/use/migrate-to-10.0.0 , https://eslint.org/docs/latest/use/migrate-to-9.0.0
