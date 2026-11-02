# Node.js API: ESLint, Linter, SourceCode

## Contents

- Choosing a class
- `ESLint` constructor options
- Linting, fixing, formatting
- `LintResult` fields
- `Linter`
- Old patterns

## Choosing a class

- Use only documented API: the rest may change at any time. Prefer the CLI unless you are building an integration. https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/use/core-concepts/
- `ESLint` reads config files and the file system, Node.js only. `Linter` reads no files and no config files and runs in browsers. https://eslint.org/docs/latest/integrate/nodejs-api
- To lint text with the project config, use `ESLint#lintText()`, not `Linter#verify()`. https://eslint.org/docs/latest/integrate/nodejs-api
- Integration supporting ESLint 10+ only: call `new ESLint()`. Use `loadESLint()` only to support several majors. https://eslint.org/docs/latest/integrate/nodejs-api
- An integration installs `eslint` as a dependency, not a dev dependency. https://eslint.org/docs/latest/integrate/integration-tutorial

## `ESLint` constructor options

| Option | Default | Note |
| :-- | :-- | :-- |
| `cwd` | `process.cwd()` | Must be absolute |
| `overrideConfigFile` | `null` (search) | `true`: no config file. A string: that file, no search |
| `overrideConfig` | `null` | Applied after the config file |
| `baseConfig` | `null` | Extended by every config |
| `errorOnUnmatchedPattern` | `true` | `lintFiles()` throws when nothing matches |
| `passOnNoPatterns` | `false` | `true`: no patterns means no failure |
| `warnIgnored` | `true` | Warns for ignored files in the list |
| `ignore`, `ignorePatterns` | `true`, `null` | Patterns relative to `cwd`, added to config ignores |
| `allowInlineConfig` | `true` | `false` overrides `noInlineConfig` |
| `fix` | `false` | Does not write files |
| `cache`, `cacheLocation`, `cacheStrategy` | `false`, `.eslintcache`, `"metadata"` | `lintText()` never caches |
| `applySuppressions`, `suppressionsLocation` | `false`, `eslint-suppressions.json` | Creating and pruning are CLI only |
| `concurrency` | `"off"` | Number or `"auto"`. Other options must be structured-cloneable |
| `flags`, `stats` | `[]`, `false` | `ESLINT_FLAGS` is read too |

https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/use/suppressions , https://eslint.org/docs/latest/pages/flags

- With `concurrency` set, every other option must be structured-cloneable. `ESLint.fromOptionsModule(url)` lifts that limit by passing the module URL to the workers. https://eslint.org/docs/latest/integrate/nodejs-api

```js
// eslint-options.js
export default { concurrency: "auto", overrideConfig: config, overrideConfigFile: true, stats: true };
// main.js
const eslint = await ESLint.fromOptionsModule(new URL("./eslint-options.js", import.meta.url));
```

https://eslint.org/docs/latest/integrate/nodejs-api

## Linting, fixing, formatting

- `fix: true` only computes fixes. Write them with `await ESLint.outputFixes(results)`, which skips files that no longer exist. https://eslint.org/docs/latest/integrate/nodejs-api
- `lintText(code, { filePath })`: without `filePath` the config for files in `cwd` applies and `result.filePath` is `"<text>"`. An ignored `filePath` returns `[]`, unless `warnIgnored` is set. Suppressions need `filePath`. https://eslint.org/docs/latest/integrate/nodejs-api
- `calculateConfigForFile(path)` rejects directories. `findConfigFile()` resolves to `undefined` with `overrideConfigFile: true`. https://eslint.org/docs/latest/integrate/nodejs-api
- `loadFormatter(name)`: none gives `stylish`, `"foo"` gives `eslint-formatter-foo`, `"@foo"` gives `@foo/eslint-formatter`, `"@foo/bar"` gives `@foo/eslint-formatter-bar`. A path must contain `/`. `format(results)` needs no 2nd argument. https://eslint.org/docs/latest/integrate/nodejs-api
- `ESLint.defaultConfig` changes between versions: do not depend on its content. https://eslint.org/docs/latest/integrate/nodejs-api

```js
const { ESLint } = require("eslint");
const eslint = new ESLint({ fix: true });
const results = await eslint.lintFiles(["lib/**/*.js"]);
await ESLint.outputFixes(results);
const formatter = await eslint.loadFormatter("stylish");
console.log(formatter.format(results));
```

```js
const eslint = new ESLint({
  overrideConfigFile: true,
  overrideConfig: { languageOptions: { ecmaVersion: 2018, sourceType: "commonjs" } },
});
const results = await eslint.lintText(testCode);
```

https://eslint.org/docs/latest/integrate/nodejs-api

## `LintResult` fields

- `output` is undefined when nothing was fixable. `source` is undefined without messages or when `output` exists. `errorCount` includes fixable and fatal errors. https://eslint.org/docs/latest/integrate/nodejs-api
- `ruleId` is `null` for core and fatal messages. `EditInfo.range` is 0-based, `line` and `column` 1-based. https://eslint.org/docs/latest/integrate/nodejs-api
- `usedDeprecatedRules` holds `{ ruleId, replacedBy, info }`, with `info` from `meta.deprecated`. With `stats: true` each result gets a `stats` key. https://eslint.org/docs/latest/integrate/nodejs-api , https://eslint.org/docs/latest/extend/stats

## `Linter`

- `verify(code, config, options)` takes a flat config. A string 3rd argument is the filename. `disableFixes: true` drops fixes and suggestions. https://eslint.org/docs/latest/integrate/nodejs-api
- `filterCodeBlock` replaces the default, which keeps only `*.js` blocks of a processor. https://eslint.org/docs/latest/integrate/nodejs-api
- `cwd` defaults to `process.cwd()` when `process` exists and reaches rules as `context.cwd`. https://eslint.org/docs/latest/integrate/nodejs-api
- Custom rules and parsers go through config: `plugins: { "my-plugin": { rules } }` and `languageOptions.parser`. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- The `SourceCode` constructor throws on an AST without locations, ranges, comments and tokens, and strips the BOM: parse the stripped text. https://eslint.org/docs/latest/integrate/nodejs-api

```js
const { Linter } = require("eslint");
const linter = new Linter({ cwd: "path/to/project" });
const messages = linter.verify("var foo;", { rules: { semi: 2 } }, { filename: "foo.js" });
const fixed = linter.verifyAndFix("var foo", { rules: { semi: 2 } });
// { fixed: true, output: "var foo;", messages: [] }
```

https://eslint.org/docs/latest/integrate/nodejs-api

## Old patterns

| Old | Current |
| :-- | :-- |
| `FlatESLint`, `LegacyESLint` | `ESLint` |
| `loadESLint({ useFlatConfig })` | Option ignored, always flat. Check `configType` |
| `new Linter({ configType: "eslintrc" })` | Flat config only |
| `linter.defineRule()`, `defineRules()`, `defineParser()`, `getRules()` | Plugins and `languageOptions.parser` in the config |
| `verify(code, { parserOptions })` | `verify(code, { languageOptions })` |
| `LintMessage.nodeType` | Removed |

https://eslint.org/docs/latest/use/migrate-to-10.0.0 , https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/integrate/nodejs-api
