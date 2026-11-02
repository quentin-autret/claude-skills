# Plugins, shareable configs, parsers, processors, languages, formatters

## Contents

- Plugin object and `meta`
- Configs inside a plugin
- Publishing a plugin
- Shareable configs
- Custom parsers
- Custom processors
- Languages
- Custom formatters
- Old patterns

## Plugin object and `meta`

- A plugin is an object `{ meta, configs, rules, processors }` (plus `languages`), exported as the package's default export so `--plugin` can import it. https://eslint.org/docs/latest/extend/plugins
- Set `meta.name` (npm name), `meta.version` and `meta.namespace` (the rule prefix). Without `meta`, `--cache` and `--print-config` do not work, and `namespace` lets `defineConfig()` find the plugin under any key the user picks. https://eslint.org/docs/latest/extend/plugins , https://eslint.org/docs/latest/extend/plugin-migration-flat-config
- Rule ids must not contain `/`. A namespace may contain `/` only if it starts with `@`. https://eslint.org/docs/latest/extend/plugins
- Names: `eslint-plugin-<name>`, `@<scope>/eslint-plugin-<name>` or `@<scope>/eslint-plugin`. https://eslint.org/docs/latest/extend/plugins

```js
import fs from "node:fs";
const pkg = JSON.parse(fs.readFileSync(new URL("./package.json", import.meta.url), "utf8"));
const plugin = {
  meta: { name: pkg.name, version: pkg.version, namespace: "example" },
  configs: {},
  rules: { "dollar-sign": dollarSignRule },
  processors: {},
};
export default plugin;
```

https://eslint.org/docs/latest/extend/plugins

## Configs inside a plugin

- Add configs after `plugin` exists so they can reference the plugin object. Rules inside use the namespace prefix. https://eslint.org/docs/latest/extend/plugins
- A config may be 1 object or an array. Users write `plugins: { example }` and `extends: ["example/recommended"]`. https://eslint.org/docs/latest/extend/plugins , https://eslint.org/docs/latest/extend/plugin-migration-flat-config
- A plugin cannot force its configs on users: they must extend them. https://eslint.org/docs/latest/extend/plugins
- Environments do not exist in flat config: export a config with `languageOptions.globals` instead. https://eslint.org/docs/latest/extend/plugin-migration-flat-config

```js
Object.assign(plugin.configs, {
  recommended: {
    plugins: { example: plugin },
    rules: { "example/dollar-sign": "error" },
  },
});
```

https://eslint.org/docs/latest/extend/plugins

## Publishing a plugin

- `"peerDependencies": { "eslint": ">=10.0.0" }`, `eslint` also as a dev dependency, keywords `eslint`, `eslintplugin`, `eslint-plugin`. https://eslint.org/docs/latest/extend/plugins , https://eslint.org/docs/latest/extend/custom-rule-tutorial
- Lint the plugin with the `recommended` configs of `eslint`, `eslint-plugin-eslint-plugin` and `eslint-plugin-n`. Test rules with `RuleTester`. https://eslint.org/docs/latest/extend/plugins
- To keep eslintrc users, ship a CommonJS entry (eslintrc cannot load ESM-only), keep `environments`, and name legacy configs `legacy-recommended`. A plugin older than 9.0.0 with a legacy `recommended` adds `flat/recommended`: `defineConfig()` falls back to it when `recommended` is not flat. https://eslint.org/docs/latest/extend/plugins , https://eslint.org/docs/latest/extend/plugin-migration-flat-config

## Shareable configs

- An npm package whose `main` exports a config object or, usually, an array. Names `eslint-config-*`, `@scope/eslint-config` or `@scope/eslint-config-name`. Keywords `eslint`, `eslintconfig`. https://eslint.org/docs/latest/extend/shareable-configs
- `peerDependencies`: `eslint` with a `>=` range on the lowest supported version. Plugins and parsers it uses go in `dependencies`. https://eslint.org/docs/latest/extend/shareable-configs
- Keep a default export. Extra configs are imported by path: `import special from "eslint-config-myconfig/my-special-config.js"`. Document an `eslint.config.js` example. https://eslint.org/docs/latest/extend/shareable-configs
- It works in `extends`, not with `--config`. https://eslint.org/docs/latest/extend/shareable-configs

```js
export default defineConfig([
  { files: ["**/*.js"], extends: [myconfig], rules: { "no-unused-vars": "warn" } },
]);
```

https://eslint.org/docs/latest/extend/shareable-configs

## Custom parsers

- An object with an own `parse(code, options)` returning the AST, or `parseForESLint(code, options)` returning `{ ast, services?, scopeManager?, visitorKeys? }`. `options` are the config `parserOptions`. Set `meta.name` and `meta.version`. https://eslint.org/docs/latest/extend/custom-parsers
- `services` reach rules as `context.sourceCode.parserServices`. https://eslint.org/docs/latest/extend/custom-parsers
- ESTree AST. Every node needs `range` (0-based, no surrounding parentheses) and a non-null `loc`, and `parent` must be writable. `Program` needs sorted, non-overlapping `tokens` and `comments`. `Literal` needs `raw`. https://eslint.org/docs/latest/extend/custom-parsers
- ESLint 10: set `Program.range` to `[0, code.length]`. A custom `ScopeManager` must resolve references to globals declared in the code and implement `addGlobals(names)`. https://eslint.org/docs/latest/use/migrate-to-10.0.0 , https://eslint.org/docs/latest/extend/custom-parsers

## Custom processors

- `{ meta, preprocess(text, filename), postprocess(messages, filename), supportsAutofix }`. `preprocess` returns `{ text, filename }` blocks. Give each block filename an extension: `files` patterns decide how the block is linted. https://eslint.org/docs/latest/extend/custom-processors
- `postprocess` receives 1 message array per block, maps locations back to the original file and returns 1 flat array. https://eslint.org/docs/latest/extend/custom-processors
- ESLint does not autofix through a processor, even with `--fix`. Set `supportsAutofix: true` and remap each `fix.range` in `postprocess`. https://eslint.org/docs/latest/extend/custom-processors
- Give both the plugin and each processor a `meta`: the plugin `meta` serves `"plugin/name"` references, the processor `meta` serves direct object references. https://eslint.org/docs/latest/extend/custom-processors
- Processor names must be identifiers (`markdown`), not extensions (`".md"`). Tell users to set `processor: "example/markdown"` with `files: ["**/*.md"]`. https://eslint.org/docs/latest/extend/plugin-migration-flat-config

## Languages

- A plugin adds a language under `languages`. Users set `language: "example/my"`. https://eslint.org/docs/latest/extend/languages
- The `Language` object implements `fileType` (`"text"`), `lineStart`, `columnStart` (0 or 1), `nodeTypeKey`, `validateLanguageOptions` (required even without options, throws on bad values, ignores unknown keys), `parse`, `createSourceCode`. Provide `visitorKeys` for non-ESTree trees. https://eslint.org/docs/latest/extend/languages
- `SourceCode` implements `TextSourceCode` from `@eslint/core` (`ast`, `text`, `getLoc()`, `getRange()`, `traverse()`). `TextSourceCodeBase` from `@eslint/plugin-kit` helps. https://eslint.org/docs/latest/extend/languages

## Custom formatters

- A formatter is `(results, context) => string`, async allowed. `context` has `cwd`, `rulesMeta`, `color` (omitted without `--color` / `--no-color`) and `maxWarningsExceeded`. https://eslint.org/docs/latest/extend/custom-formatters
- Formatters take no CLI arguments: read environment variables, or pipe `-f json` into your program. https://eslint.org/docs/latest/extend/custom-formatters
- Print `file:line:column` so terminals make it clickable. Publish as `eslint-formatter-*` with keywords `eslint`, `eslint-formatter`, `eslintformatter`. https://eslint.org/docs/latest/extend/custom-formatters
- Write a formatter only when the built-ins do not fit. https://eslint.org/docs/latest/extend/ways-to-extend

## Old patterns

| Old | Current |
| :-- | :-- |
| `name`, `version` at the plugin root | `meta.name`, `meta.version` |
| Configs with `plugins: ["example"]` | `plugins: { example: plugin }` |
| Processor named `".md"`, applied by extension | Named `markdown`, set by the user |
| `environments` | Configs with `languageOptions.globals` |
| `parser: "eslint-parser-myparser"` | `languageOptions: { parser: myparser }` |
| `CLIEngine` formatter context | `ESLint` class context |

https://eslint.org/docs/latest/extend/plugins , https://eslint.org/docs/latest/extend/plugin-migration-flat-config , https://eslint.org/docs/latest/extend/custom-parsers , https://eslint.org/docs/latest/extend/custom-formatters
