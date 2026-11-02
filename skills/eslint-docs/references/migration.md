# Migrating to ESLint 9 and 10, eslintrc to flat config

## Contents

- Before upgrading
- Codemods and converters
- eslintrc to flat config
- ESLint 10 changes for users
- ESLint 9 changes for users
- Rule option changes
- Plugins that break after the upgrade
- Old patterns

## Before upgrading

- ESLint 10 needs Node.js `^20.19.0`, `^22.13.0` or `>=24`. Check the Node version of editor integrations too. If you cannot upgrade Node, stay on ESLint 9. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- A TypeScript config needs `jiti` >= 2.2.0 in ESLint 10. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- vscode-eslint supports ESLint 9 from 3.0.10. https://eslint.org/docs/latest/use/configure/migration-guide

## Codemods and converters

- 9 to 10: `npx codemod @eslint/v9-to-v10`, then review against the breaking changes list. Sub-codemods: `@eslint/v9-to-v10-config`, `-custom-rules`, `-ruletester`, `-linter-api`. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- 8 to 9: `npx codemod @eslint/v8-to-v9-config` and `npx codemod @eslint/v8-to-v9-custom-rules`. Point the rules codemod at rule files only: it rewrites any JS or TS file it reaches. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `npx @eslint/migrate-config .eslintrc.json` converts an eslintrc file. The output is a starting point. With `.eslintrc.js` it keeps only the evaluated data, without functions or conditionals. https://eslint.org/docs/latest/use/configure/migration-guide

## eslintrc to flat config

- eslintrc is removed in ESLint 10: setting `ESLINT_USE_FLAT_CONFIG` to `false` no longer brings it back. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- Every flat config object behaves like an eslintrc `overrides` entry. An object without `files` targets `"**/*.{js,mjs,cjs}"` plus whatever other objects match. https://eslint.org/docs/latest/use/configure/migration-guide
- `root` does not exist (always root). `files` must be an array. `package.json` `eslintConfig` is not read. https://eslint.org/docs/latest/use/configure/migration-guide
- `env` is gone: use `languageOptions.globals` with the `globals` package, which ESLint does not install. `parserOptions` moves under `languageOptions`. https://eslint.org/docs/latest/use/configure/migration-guide
- `noInlineConfig` and `reportUnusedDisableDirectives` move into `linterOptions`. https://eslint.org/docs/latest/use/configure/migration-guide
- Plugins and parsers are imported objects, not strings. Processors are never applied automatically: `{ files: ["**/*.md"], processor: somePlugin.processors[".md"] }`. https://eslint.org/docs/latest/use/configure/migration-guide
- A shareable eslintrc config without a flat version: `FlatCompat` from `@eslint/eslintrc`, `new FlatCompat({ baseDirectory: __dirname })`, then `...compat.extends("eslint-config-my-config")`. Upgrade `@eslint/eslintrc` to latest when using `@eslint/js` v10 with `FlatCompat`. https://eslint.org/docs/latest/use/configure/migration-guide , https://eslint.org/docs/latest/use/migrate-to-10.0.0
- Unchanged: rule config syntax, processor syntax, the global variable concept and most of the CLI. https://eslint.org/docs/latest/use/configure/migration-guide

```js
import { defineConfig } from "eslint/config";
import js from "@eslint/js";
import globals from "globals";

export default defineConfig([
  js.configs.recommended,
  { files: ["**/*.js"], languageOptions: { globals: { ...globals.browser, ...globals.node } } },
  { ignores: ["**/temp.js", "config/*"] },
]);
```

https://eslint.org/docs/latest/use/configure/migration-guide

## ESLint 10 changes for users

- Config lookup starts from each linted file's directory, not the cwd. To keep a single root config for everything, pass `--config path/to/eslint.config.js`. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `/* eslint-env */` comments are reported as errors. Replace them with `/* global describe, it */` or config globals. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `js/recommended` adds `no-unassigned-vars`, `no-useless-assignment`, `preserve-caught-error`. Fix or disable. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- JSX references are tracked in scope analysis: expect new `no-unused-vars` and `no-undef` reports in JSX files. Remove workaround rules such as `@eslint-react/jsx-uses-vars`. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `stylish` uses `util.styleText`: it honors `NO_COLOR` and `NODE_DISABLE_COLORS`, and `--color` / `--no-color` win over environment variables. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `@eslint/js` configs now carry a `name`. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `LintMessage.nodeType` is removed: update custom formatters. https://eslint.org/docs/latest/use/migrate-to-10.0.0

## ESLint 9 changes for users

- `eslint.config.js` is the default and eslintrc is not searched. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `--quiet` stops running `"warn"` rules. A warn rule that feeds others (one calling `markVariableAsUsed()`) silently stops working: set it to `"error"` or drop `--quiet`. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `npx eslint` with no argument lints `.`. `--pass-on-no-patterns` restores exit 0. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- A severity-only `/* eslint rule: "warn" */` keeps the configured options, and duplicate rule comments are errors (the first wins). https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `/* exported foo, bar */` is the only valid form. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `js/recommended` added `no-constant-binary-expression`, `no-empty-static-block`, `no-new-native-nonconstructor`, `no-unused-private-class-members`. It dropped `no-extra-semi`, `no-inner-declarations`, `no-mixed-spaces-and-tabs`, `no-new-symbol`. https://eslint.org/docs/latest/use/migrate-to-9.0.0

## Rule option changes

| Rule | Change | Fix |
| :-- | :-- | :-- |
| `no-unused-vars` (9) | `caughtErrors` defaults to `"all"` | `catch {` without binding, or `"none"` |
| `no-unused-vars` (9) | `varsIgnorePattern` skips catch params | `caughtErrorsIgnorePattern` |
| `no-useless-computed-key` (9) | `enforceForClassMembers: true` | Set `false` to restore |
| `no-implicit-coercion` (9) | Reports `-(-foo)`, `foo - 0` | `"allow": ["-", "- -"]` |
| `no-restricted-imports` (9) | All `paths` entries with one `name` apply | Review entries sharing a `name` |
| `no-invalid-regexp` (9, 10) | `allowConstructorFlags` case-sensitive, no duplicates | Clean the list |
| `no-constructor-return`, `no-sequences`, `camelcase` (9) | Invalid options rejected | `camelcase` `allow` is a string array |
| `radix` (10) | `"always"`, `"as-needed"` do nothing | Remove `"always"`, pass the radix |
| `no-shadow-restricted-names` (10) | `reportGlobalThis: true` | `{ "reportGlobalThis": false }` |
| `func-names` (10) | Extra array items rejected | Base string plus `{ generators }` |

https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/use/migrate-to-10.0.0

- `require-jsdoc` and `valid-jsdoc` are removed (9). Use `eslint-plugin-jsdoc`. https://eslint.org/docs/latest/use/migrate-to-9.0.0

## Plugins that break after the upgrade

- `TypeError: context.getScope is not a function` (or `getAncestors`, `markVariableAsUsed`, `getDeclaredVariables`) means the plugin still uses the pre-9 rule API. Upgrade the plugin. If already on its latest version, patch it in your config with ESLint's compatibility utilities and open an issue on the plugin. https://eslint.org/docs/latest/use/troubleshooting/v9-rule-api-changes
- Compatibility patches for the removed `SourceCode` methods live in `@eslint/compat`. https://eslint.org/docs/latest/use/migrate-to-10.0.0

## Old patterns

| Old | Current |
| :-- | :-- |
| `.eslintrc.*`, `eslintConfig` in `package.json` | `eslint.config.js` with `defineConfig()` |
| `"extends": "eslint:recommended"` | `js.configs.recommended`, or `extends: ["js/recommended"]` |
| `env: { browser: true }` | `languageOptions: { globals: globals.browser }` |
| `/* eslint-env mocha */` | `/* global describe, it */` or config globals |
| `parser: "string"` | `languageOptions: { parser: importedObject }` |
| `overrides: [...]` | More config objects with `files` |
| `.eslintignore`, `ignorePatterns` | Global ignores |
| `ESLINT_USE_FLAT_CONFIG` | Not needed, flat config only |
| `FlatESLint`, `LegacyESLint` | `ESLint` |
| `FlatRuleTester` | `RuleTester` |

https://eslint.org/docs/latest/use/configure/migration-guide , https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/use/migrate-to-10.0.0 , https://eslint.org/docs/latest/use/configure/configuration-files
