---
name: eslint-docs
description: Teaches current ESLint 10 practices drawn from the official documentation. Use when writing, reviewing or fixing an eslint.config.js (flat config, defineConfig, files, ignores, globalIgnores, extends, languageOptions, globals, parsers, processors), configuring ESLint rules and severities, disable comments and inline config, running the CLI in CI (--fix, --cache, --quiet, --max-warnings, bulk suppressions, formatters, feature flags, MCP server), writing custom rules (meta, schema, fixes, suggestions, selectors, scope, code paths) and testing them with RuleTester, building plugins, shareable configs, custom parsers, processors or formatters, using the Node.js API (ESLint, Linter), or migrating to ESLint 9 or 10 from eslintrc. Triggers on eslint, eslint.config.js, flat config, .eslintrc, .eslintignore, @eslint/js, js/recommended, eslint-disable, no-unused-vars, no-restricted-imports, no-restricted-syntax, RuleTester, eslint-plugin, eslint-config, FlatCompat, context.getScope is not a function, ESLint 10.
metadata:
  version: "ESLint 10.14.0 (docs from the latest tag)"
  source: https://eslint.org/docs/latest/
  fetched: "2026-11-02"
---

# ESLint 10

This skill covers ESLint 10.14.0 as documented on https://eslint.org/docs/latest/. It holds what differs from habit: flat config pitfalls, surprising defaults, removed APIs and the patterns the docs recommend. Each claim in the reference files ends with the page it comes from.

## How to navigate

1. Read the quick rules below. They catch the most frequent mistakes.
2. Open the one reference file that matches the task, from the table at the end.
3. When the code uses `.eslintrc`, `env`, `context.getScope()` or `FlatESLint`, read `references/migration.md` first.

## Quick start

```sh
npm init @eslint/config@latest
```

```js
// eslint.config.js
import js from "@eslint/js";
import globals from "globals";
import { defineConfig, globalIgnores } from "eslint/config";

export default defineConfig([
  globalIgnores(["dist/"]),
  {
    files: ["**/*.js", "**/*.mjs", "**/*.cjs"],
    plugins: { js },
    extends: ["js/recommended"],
    languageOptions: { globals: { ...globals.browser } },
    rules: { "no-unused-vars": "warn" },
  },
]);
```

https://eslint.org/docs/latest/use/getting-started , https://eslint.org/docs/latest/use/configure/configuration-files

- Node.js `^20.19.0`, `^22.13.0` or `>=24`. TypeScript types need TypeScript 5.3+. https://eslint.org/docs/latest/use/getting-started
- Install `eslint` and `@eslint/js` locally as dev dependencies. A global install is not recommended and plugins must still be local. `globals` is a separate package. https://eslint.org/docs/latest/use/getting-started , https://eslint.org/docs/latest/use/configure/migration-guide
- `eslint.config.ts` needs the dev dependency `jiti` >= 2.2.0 on Node. https://eslint.org/docs/latest/use/configure/configuration-files

## Quick rules

### Config files

- ESLint lints nothing useful until you extend a config or enable rules. https://eslint.org/docs/latest/use/getting-started
- Config lookup starts in each linted file's directory and walks up. `--config` forces 1 file. https://eslint.org/docs/latest/use/configure/configuration-files , https://eslint.org/docs/latest/use/migrate-to-10.0.0
- An object without `files` applies to every file another object matches. Once you lint `.ts`, `.md` or `.json`, scope JS rules with `files`, and always put `files` next to `extends`. https://eslint.org/docs/latest/use/configure/configuration-files
- `ignores` alone in an object (global ignores) can match directories. Next to other keys it matches files only: `"dir/**"`, not `"dir/"`. Prefer `globalIgnores()`. https://eslint.org/docs/latest/use/configure/configuration-files
- Dotfiles are linted unless ignored. Only `**/node_modules/` and `.git/` are ignored by default. https://eslint.org/docs/latest/use/configure/ignore , https://eslint.org/docs/latest/use/configure/configuration-files
- Import `.gitignore` with `includeIgnoreFile(path, { gitignoreResolution: true })` from `eslint/config`. https://eslint.org/docs/latest/use/configure/ignore
- No environments: declare globals with the `globals` package in `languageOptions.globals`. https://eslint.org/docs/latest/use/configure/language-options
- Plugins, parsers and processors are imported objects. Processors are never applied automatically. https://eslint.org/docs/latest/use/configure/migration-guide
- Inspect what matches a file with `npx eslint --inspect-config` or `--print-config file.js`. https://eslint.org/docs/latest/use/configure/debug

### Rules and comments

- A severity-only override keeps the earlier options, in config and in `/* eslint */` comments. https://eslint.org/docs/latest/use/configure/rules , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- An inline comment cannot load a plugin. A bare `/* eslint-enable */` re-enables every rule. https://eslint.org/docs/latest/use/configure/rules
- Put the reason after `--`: `// eslint-disable-next-line no-console -- reason`. https://eslint.org/docs/latest/use/configure/rules
- `/* eslint-env */` is an error in ESLint 10. Use `/* global name */` or config globals. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `no-unused-vars` reports unused `catch (e)` by default: write `catch {`. https://eslint.org/docs/latest/rules/no-unused-vars
- Do not enable `no-return-await`. Use a formatter or ESLint Stylistic rather than core layout rules. https://eslint.org/docs/latest/rules/no-return-await , https://eslint.org/docs/latest/use/core-concepts/glossary

### CLI and CI

- Exit code 1 means errors (or too many warnings), 2 means a config or internal error. https://eslint.org/docs/latest/use/command-line-interface
- `--quiet` does not run `"warn"` rules, unless `--max-warnings` is also set: then they run but are not reported. https://eslint.org/docs/latest/use/command-line-interface
- Quote globs. `--fix` throws on stdin: use `--fix-dry-run --format json`. https://eslint.org/docs/latest/use/command-line-interface
- `--cache` ignores cross-file dependencies: type-aware and import rules can be stale. https://eslint.org/docs/latest/use/command-line-interface
- Adopt a strict rule on old code with `eslint --fix --suppress-all` and commit `eslint-suppressions.json`. https://eslint.org/docs/latest/use/suppressions

### Custom rules and integrations

- A rule is `{ meta, create }`. `meta.fixable` and `meta.hasSuggestions` are required to fix or suggest, `meta.schema` to accept options. https://eslint.org/docs/latest/extend/custom-rules
- Use `context.sourceCode`, `context.filename` and `sourceCode.getScope(node)`. The `context.getX()` methods are gone. https://eslint.org/docs/latest/use/migrate-to-10.0.0 , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `new ESLint({ fix: true })` writes nothing until `ESLint.outputFixes(results)`. https://eslint.org/docs/latest/integrate/nodejs-api
- Give a plugin `meta.name`, `meta.version` and `meta.namespace` for easier debugging and more effective caching. The namespace lets `defineConfig()` find the plugin under another name. https://eslint.org/docs/latest/extend/plugins

## Old patterns

| Old | Current |
| :-- | :-- |
| `.eslintrc.*`, `eslintConfig` in `package.json` | `eslint.config.js` with `defineConfig()` |
| `"extends": "eslint:recommended"` | `extends: ["js/recommended"]` with `plugins: { js }` |
| `env`, `/* eslint-env */` | `languageOptions.globals`, `/* global */` |
| `.eslintignore`, `--ignore-path` | `globalIgnores()`, `includeIgnoreFile()` |
| `overrides` | More config objects with `files` |
| `--rulesdir`, `linter.defineRule()` | Inline plugin `plugins: { local: { rules } }` |
| `--no-eslintrc` | `--no-config-lookup` |
| `context.getScope()`, `context.getSourceCode()` | `sourceCode.getScope(node)`, `context.sourceCode` |
| Function-style rules | `{ meta, create }` |
| `FlatESLint`, `LegacyESLint`, `FlatRuleTester` | `ESLint`, `RuleTester` |
| `require-jsdoc`, `valid-jsdoc` | `eslint-plugin-jsdoc` |

Details and the full lists: `references/migration.md`.

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/configuration.md](references/configuration.md) | Config file names and lookup, TypeScript configs, config object keys, `files` matching, `extends`, language options, globals, parsers, processors, languages, debugging |
| [references/ignoring-files.md](references/ignoring-files.md) | Global and non-global `ignores`, default ignores, dotfiles, unignore recipes, `.gitignore` import |
| [references/rules-and-inline-config.md](references/rules-and-inline-config.md) | Severities, option merging, inline config comments, disable directives, `linterOptions`, fixes versus suggestions, circular fixes |
| [references/cli.md](references/cli.md) | Globs, `--fix`, exit codes, `--quiet`, formatters, cache, bulk suppressions, feature flags, concurrency, MCP server |
| [references/migration.md](references/migration.md) | ESLint 9 and 10 breaking changes, codemods, eslintrc to flat config, changed rule options, broken plugins |
| [references/custom-rules.md](references/custom-rules.md) | Rule `meta`, local rules, `context.report`, fixes, suggestions, `meta.schema`, `meta.defaultOptions`, deprecation, removed `context` methods |
| [references/ast-scope-code-paths.md](references/ast-scope-code-paths.md) | Selectors, `SourceCode` methods, scopes, scope manager objects, code path events, profiling |
| [references/testing-rules.md](references/testing-rules.md) | `RuleTester` defaults, valid and invalid cases, fix and suggestion assertions, strict checks, framework hooks |
| [references/plugins-and-shareable-configs.md](references/plugins-and-shareable-configs.md) | Plugin object and `meta`, plugin configs, publishing, shareable configs, custom parsers, processors, languages, formatters |
| [references/nodejs-api.md](references/nodejs-api.md) | `ESLint` options and methods, `lintText`, `outputFixes`, `loadFormatter`, `LintResult`, `Linter`, `SourceCode` |
| [references/core-rules-problems.md](references/core-rules-problems.md) | Rules TypeScript covers, unused and undeclared variables, shadowing, conditions, async errors, regex traps |
| [references/core-rules-restrictions.md](references/core-rules-restrictions.md) | `no-restricted-imports`, `-globals`, `-properties`, `-exports`, `no-restricted-syntax` recipes, other bans, size and complexity limits |
| [references/core-rules-style.md](references/core-rules-style.md) | Suggestion rule defaults, layout rule defaults, removed rules, deprecated options |

Crawled pages: [SOURCES.md](SOURCES.md).
