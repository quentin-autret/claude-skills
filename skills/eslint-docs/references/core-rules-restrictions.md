# Core rules: restrictions and size limits

## Contents

- `no-restricted-imports`
- `no-restricted-globals`, `no-restricted-properties`, `no-restricted-exports`
- `no-restricted-syntax` recipes
- Other configurable bans
- Size and complexity limits

## `no-restricted-imports`

- Checks static imports and `export ... from`, never dynamic `import()`. https://eslint.org/docs/latest/rules/no-restricted-imports
- `paths` match exact module names: a `paths` entry for `import-foo` does not cover `import-foo/bar`. Use `patterns` for sub-paths. https://eslint.org/docs/latest/rules/no-restricted-imports
- `patterns` entries use gitignore-style `group` or a `regex`, never both. `caseSensitive` defaults to `false`. https://eslint.org/docs/latest/rules/no-restricted-imports
- A negation `!pattern` comes last and cannot re-include a file whose parent directory is excluded: re-include the directory (`"!import1/private"`). https://eslint.org/docs/latest/rules/no-restricted-imports
- Name filters: `importNames`, `importNamePattern`, or the allow-forms `allowImportNames`, `allowImportNamePattern`, which cannot be combined with the deny-forms or with each other (in `paths`, `importNames` and `allowImportNames` are exclusive). `"default"` in `importNames` targets the default import. `importNamePattern: "^"` allows only side-effect imports. https://eslint.org/docs/latest/rules/no-restricted-imports
- TypeScript: `allowTypeImports: true` lets type-only imports through. `import x = require()` can only be banned as a whole. https://eslint.org/docs/latest/rules/no-restricted-imports
- Since ESLint 9, several `paths` entries with the same `name` all apply. https://eslint.org/docs/latest/use/migrate-to-9.0.0

```json
"no-restricted-imports": ["error", {
  "paths": [{ "name": "import-foo", "importNames": ["Bar"], "message": "Please use Bar from /import-bar/baz/ instead." }],
  "patterns": ["import1/private/*", "import2/*", "!import2/good"]
}]
```

https://eslint.org/docs/latest/rules/no-restricted-imports

- CommonJS `require()` is `no-restricted-modules`: custom messages work for `paths` only. https://eslint.org/docs/latest/rules/no-restricted-modules

## `no-restricted-globals`, `no-restricted-properties`, `no-restricted-exports`

- `no-restricted-globals` takes names or `{ name, message }`. The object form adds `checkGlobalObject` (default `false`) and `globalObjects` (`globalThis`, `self`, `window`). TypeScript type positions are ignored. https://eslint.org/docs/latest/rules/no-restricted-globals
- `no-restricted-properties` entries are `{ object, property, message }`. Omit `object` to ban a property on every object, omit `property` to ban every property of an object. `allowObjects` cannot be used with `object`, `allowProperties` cannot be used with `property`. It covers dot access and destructuring. https://eslint.org/docs/latest/rules/no-restricted-properties
- `no-restricted-exports` bans nothing by default. `"default"` in `restrictedNamedExports` does not ban `export default`: use `restrictDefaultExports`. `export * from "mod"` is not checked. https://eslint.org/docs/latest/rules/no-restricted-exports

```json
"no-restricted-properties": [2, { "property": "push", "allowObjects": ["router"], "message": "Prefer [...array, newValue] because it does not mutate the array in place." }]
```

https://eslint.org/docs/latest/rules/no-restricted-properties

## `no-restricted-syntax` recipes

- Takes node types or AST selectors, as strings or `{ selector, message }`. It works for any language ESLint lints. Find node types in https://explorer.eslint.org. https://eslint.org/docs/latest/rules/no-restricted-syntax
- Ban every comma operator, `for` included: `"no-restricted-syntax": ["error", "SequenceExpression"]`. https://eslint.org/docs/latest/rules/no-sequences
- Ban all template literals: the `TemplateLiteral` selector (`quotes` with `allowTemplateLiterals: false` does not). https://eslint.org/docs/latest/rules/quotes
- Report only calls to `console` methods outside an allowed list, with `no-console` turned off: https://eslint.org/docs/latest/rules/no-console

```json
{
  "rules": {
    "no-console": "off",
    "no-restricted-syntax": ["error", {
      "selector": "CallExpression[callee.object.name='console'][callee.property.name!=/^(log|warn|error|info|trace)$/]",
      "message": "Unexpected property on console object was called"
    }]
  }
}
```

```json
"no-restricted-syntax": ["error", "FunctionExpression", "WithStatement", "BinaryExpression[operator='in']"]
```

https://eslint.org/docs/latest/rules/no-restricted-syntax

- `setTimeout` with the wrong argument count: `"CallExpression[callee.name='setTimeout'][arguments.length!=2]"`. https://eslint.org/docs/latest/extend/selectors

## Other configurable bans

- `no-console` `allow` lists permitted methods: `{ "allow": ["warn", "error"] }`. Most Node.js projects do not want the rule. https://eslint.org/docs/latest/rules/no-console
- `id-denylist` lists the denied names after the severity. https://eslint.org/docs/latest/rules/id-denylist
- `no-warning-comments` defaults to `terms: ["todo", "fixme", "xxx"]`, `location: "start"`, whole words, case-insensitive. Expect a flood on an old codebase. https://eslint.org/docs/latest/rules/no-warning-comments
- `no-bitwise` takes `allow` and `int32Hint` (allows `|0`). https://eslint.org/docs/latest/rules/no-bitwise
- `no-eval` reports every `eval()` call. `allowIndirect` allows indirect eval. https://eslint.org/docs/latest/rules/no-eval
- `no-extend-native` and `no-global-assign` take `exceptions`. Disable `no-extend-native` for polyfills. https://eslint.org/docs/latest/rules/no-extend-native , https://eslint.org/docs/latest/rules/no-global-assign
- `no-sync` matches the `Sync` suffix. `allowAtRootLevel` defaults to `false`. https://eslint.org/docs/latest/rules/no-sync
- `no-duplicate-imports` flags `import type` next to a value import from the same module unless `allowSeparateTypeImports: true`. `includeExports` adds `export ... from`. https://eslint.org/docs/latest/rules/no-duplicate-imports

## Size and complexity limits

| Rule | Default |
| :-- | :-- |
| `complexity` | `max: 20`, `variant: "classic"` (`"modified"` counts a `switch` as 1) |
| `max-depth` | 4 |
| `max-len` | `code: 80`, `tabWidth: 4` |
| `max-lines` | 300 |
| `max-lines-per-function` | 50 |
| `max-nested-callbacks` | 10 |
| `max-params` | 3 |
| `max-statements` | 10 |
| `max-classes-per-file` | 1 |
| `id-length` | `min: 2`, counts graphemes |

https://eslint.org/docs/latest/rules/complexity , https://eslint.org/docs/latest/rules/max-depth , https://eslint.org/docs/latest/rules/max-len , https://eslint.org/docs/latest/rules/max-lines , https://eslint.org/docs/latest/rules/max-lines-per-function , https://eslint.org/docs/latest/rules/max-nested-callbacks , https://eslint.org/docs/latest/rules/max-params , https://eslint.org/docs/latest/rules/max-statements , https://eslint.org/docs/latest/rules/max-classes-per-file , https://eslint.org/docs/latest/rules/id-length

- For `complexity`, class field initializers and static blocks are measured apart from the enclosing code. For `max-statements`, static blocks are not counted in the enclosing function. https://eslint.org/docs/latest/rules/complexity , https://eslint.org/docs/latest/rules/max-statements
- A nested long method chain counts as 1 statement and complexity 1: only `max-lines-per-function` catches it. https://eslint.org/docs/latest/rules/max-lines-per-function
- Escape `max-len` `ignorePattern` twice in JSON or YAML. https://eslint.org/docs/latest/rules/max-len
