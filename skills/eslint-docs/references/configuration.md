# Configuration: eslint.config.js

## Contents

- Config file names and lookup
- Config object keys and defaults
- Matching files with `files`
- `extends`, predefined and shareable configs
- Language options and globals
- Parsers, processors, languages
- Debugging a config

## Config file names and lookup

- Names: `eslint.config.js`, `.mjs`, `.cjs`, `.ts`, `.mts`, `.cts`. The file exports an array of config objects, wrapped in `defineConfig()` from `eslint/config`. https://eslint.org/docs/latest/use/configure/configuration-files
- With `"type": "commonjs"` in `package.json`, `eslint.config.js` must be CommonJS: `const { defineConfig } = require("eslint/config")` and `module.exports = defineConfig([...])`. https://eslint.org/docs/latest/use/configure/configuration-files
- Several config files in one directory: `.js` wins over `.mjs`, `.cjs`, then the TypeScript names. https://eslint.org/docs/latest/use/configure/configuration-files
- Lookup starts in the directory of each linted file and walks up, so monorepo packages can carry their own config. `-c path/to/eslint.config.js` disables the search and uses that file only. https://eslint.org/docs/latest/use/configure/configuration-files
- TypeScript config files: Deno and Bun load them natively. Node needs the dev dependency `jiti` >= 2.2.0, which is not installed for you. ESLint does not type-check the config and ignores `tsconfig.json`. https://eslint.org/docs/latest/use/configure/configuration-files
- Node >= 22.13.0 can load a TypeScript config without `jiti` (experimental): `npx --node-options='--experimental-strip-types' eslint --flag unstable_native_nodejs_ts_config`. https://eslint.org/docs/latest/use/configure/configuration-files
- pnpm: add `auto-install-peers=true` and `node-linker=hoisted` to `.npmrc`. https://eslint.org/docs/latest/use/getting-started
- Do not install ESLint globally: plugins and shareable configs must be local anyway. https://eslint.org/docs/latest/use/getting-started

## Config object keys and defaults

Keys: `name`, `basePath`, `files`, `ignores`, `extends`, `language`, `languageOptions`, `linterOptions`, `processor`, `plugins`, `rules`, `settings`. https://eslint.org/docs/latest/use/configure/configuration-files

| Setting | Default |
| :-- | :-- |
| `language` | `"js/js"` |
| `languageOptions.ecmaVersion` | `"latest"` |
| `languageOptions.sourceType` | `"module"` for `.js` and `.mjs`, `"commonjs"` for `.cjs` |
| `linterOptions.reportUnusedDisableDirectives` | `"warn"` |
| `linterOptions.reportUnusedInlineConfigs` | `"off"` |

https://eslint.org/docs/latest/use/configure/configuration-files

- `name` is optional but recommended, scoped with `/` (`example/recommended`). Keep names unique, ESLint does not check. https://eslint.org/docs/latest/use/configure/configuration-files
- `basePath` scopes an object to a subdirectory: its `files` and `ignores` become relative to it. https://eslint.org/docs/latest/use/configure/configuration-files
- `settings` reaches every rule as `context.settings`. Plugins namespace their own keys. https://eslint.org/docs/latest/use/configure/configuration-files
- All objects matching a file merge in order, the later one wins on conflict. `languageOptions.globals` merge. https://eslint.org/docs/latest/use/configure/configuration-files

## Matching files with `files`

- Linted by default: `**/*.js`, `**/*.cjs`, `**/*.mjs`. Other extensions are linted only when some `files` pattern matches them. https://eslint.org/docs/latest/use/configure/configuration-files
- An object without `files` applies to every file any other object matches. Once you lint `.ts`, `.md` or `.json` too, put JS-only rules in an object with `files: ["**/*.js", "**/*.cjs", "**/*.mjs"]`. https://eslint.org/docs/latest/use/configure/configuration-files
- `files` must be an array. Patterns use minimatch, relative to the config file (relative to the cwd when `--config` is used). https://eslint.org/docs/latest/use/configure/configuration-files , https://eslint.org/docs/latest/use/configure/migration-guide
- A pattern that is `*` or ends in `/*` or `/**` does not trigger linting of extra files. https://eslint.org/docs/latest/use/configure/configuration-files
- Files without extension: `"**/!(*.*)"`. A dotfile like `.gitignore` has the extension `gitignore`: match `"**/.gitignore"`, not `"**/*.gitignore"`. https://eslint.org/docs/latest/use/configure/configuration-files
- AND match with a nested array: `files: [["src/*", "**/*.js"]]`. https://eslint.org/docs/latest/use/configure/configuration-files
- POSIX classes work in globs since v10 (`**/[[:upper:]]*.js`). Check old patterns that look like them. https://eslint.org/docs/latest/use/migrate-to-10.0.0

```js
export default defineConfig([
  { files: ["**/*.ts", "**/*.cts", "**/*.mts"], rules: { "no-console": "error" } },
  { files: ["**/*-test.js", "**/*.spec.js"], rules: { "no-unused-expressions": "off" } },
]);
```

https://eslint.org/docs/latest/use/configure/configuration-files , https://eslint.org/docs/latest/use/configure/rules

## `extends`, predefined and shareable configs

- `extends` takes config objects, config arrays or `"namespace/configName"` strings. The string form needs the plugin in `plugins` of the same object. https://eslint.org/docs/latest/use/configure/configuration-files
- Always set `files` next to `extends`, otherwise the extended config may hit every file. Overrides in the same object, after `extends`, win. https://eslint.org/docs/latest/use/configure/configuration-files , https://eslint.org/docs/latest/extend/shareable-configs
- `js/recommended` comes from `@eslint/js`. `js/all` enables every core rule and changes with every minor: not for production. https://eslint.org/docs/latest/use/configure/configuration-files
- Use `extends` for reuse (plugins, shareable configs). Use later objects for per-file rules, severities and per-directory options. https://eslint.org/docs/latest/use/configure/configuration-files
- A shareable config cannot be passed to `--config`. https://eslint.org/docs/latest/extend/shareable-configs

```js
import js from "@eslint/js";
import { defineConfig } from "eslint/config";
export default defineConfig([
  { files: ["**/*.js"], plugins: { js }, extends: ["js/recommended"], rules: { "no-unused-vars": "warn" } },
]);
```

https://eslint.org/docs/latest/use/configure/configuration-files

## Language options and globals

- `sourceType`: `"module"` (strict, module scope, invalid with `ecmaVersion` 3 or 5), `"commonjs"` (function scope at top level), `"script"` (shared global scope). Keep `ecmaVersion: "latest"`, which also decides the built-in globals. https://eslint.org/docs/latest/use/configure/language-options
- Default parser options, all `false`: `allowReserved`, `ecmaFeatures.globalReturn` (only for `script`), `ecmaFeatures.impliedStrict`, `ecmaFeatures.jsx`. Do not put `ecmaVersion` or `sourceType` in `parserOptions`. https://eslint.org/docs/latest/use/configure/language-options
- `ecmaFeatures.jsx` parses JSX, it is not React support. Use `eslint-plugin-react` or `@eslint-react/eslint-plugin`. https://eslint.org/docs/latest/use/configure/language-options
- ESLint assumes no environment. Declare globals with the `globals` package, which you install yourself. https://eslint.org/docs/latest/use/configure/language-options , https://eslint.org/docs/latest/use/configure/migration-guide
- Global values: `"writable"`, `"readonly"`, `"off"`. In a file: `/* global var1:writable, var2 */`. https://eslint.org/docs/latest/use/configure/language-options

```js
import globals from "globals";
export default defineConfig([
  { files: ["**/*.js"], languageOptions: { globals: { ...globals.browser, myVar: "readonly" } } },
]);
```

https://eslint.org/docs/latest/use/configure/language-options

## Parsers, processors, languages

- Plugins, parsers and processors are imported objects: `plugins: { jsdoc }`, `languageOptions: { parser }`. The namespace is the key you choose (`jsd: jsdoc` works). https://eslint.org/docs/latest/use/configure/plugins , https://eslint.org/docs/latest/use/configure/migration-guide
- Inline plugin for local rules: `plugins: { local: { rules: { "my-rule": myRule } } }`, then `"local/my-rule": "error"`. https://eslint.org/docs/latest/use/configure/plugins
- The default parser is Espree. Third-party parsers (`@typescript-eslint/parser`, `@babel/eslint-parser`) are not guaranteed: ESLint does not fix bugs that affect only them. https://eslint.org/docs/latest/use/configure/parser
- Processors are never applied automatically. Set `processor: "markdown/markdown"` with `files: ["**/*.md"]`. Code blocks become child files, configured with `files: ["**/*.md/*.js"]`. https://eslint.org/docs/latest/use/configure/plugins , https://eslint.org/docs/latest/use/configure/migration-guide
- Languages: `language: "json/jsonc"` with `@eslint/json`. `languageOptions` then follow that language. https://eslint.org/docs/latest/use/configure/plugins
- A plugin that fails often still uses the pre-9 rule API or ships no flat config. Upgrade it, or patch it in your config with ESLint's compatibility utilities. https://eslint.org/docs/latest/use/configure/plugins , https://eslint.org/docs/latest/use/troubleshooting/v9-rule-api-changes

```js
import babelParser from "@babel/eslint-parser";
export default defineConfig([{ files: ["**/*.js", "**/*.mjs"], languageOptions: { parser: babelParser } }]);
```

https://eslint.org/docs/latest/use/configure/parser

## Debugging a config

- `npx eslint --debug file.js` shows which config file loads. https://eslint.org/docs/latest/use/configure/debug
- `npx eslint --print-config file.js` prints the computed config, without `files`, `ignores` and `name`. https://eslint.org/docs/latest/use/configure/debug
- `npx eslint --inspect-config` opens `@eslint/config-inspector`: which objects match a file, and deprecated rules. https://eslint.org/docs/latest/use/configure/debug
