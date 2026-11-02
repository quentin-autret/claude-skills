# Ignoring files

## Contents

- Global ignores versus non-global `ignores`
- Default ignores and dotfiles
- Pattern recipes
- Importing `.gitignore`
- CLI and warnings
- Old patterns

## Global ignores versus non-global `ignores`

- A config object whose only key is `ignores` (`name` allowed) holds global ignores: they apply to every object and can match directories. https://eslint.org/docs/latest/use/configure/configuration-files
- `ignores` next to other keys is non-global: it only matches file names, so `"dir/"` ignores nothing there. Write `"dir/**"`. https://eslint.org/docs/latest/use/configure/configuration-files
- `ignores` with other keys and no `files` acts like `files: ["**/*"]`. Always add `files` when you use `ignores`. https://eslint.org/docs/latest/use/configure/configuration-files
- A negated `ignores` entry does not make a file linted when `files` is absent. https://eslint.org/docs/latest/use/configure/configuration-files
- Prefer `globalIgnores()` from `eslint/config` to make the intent explicit. Its 2nd argument is the config name. https://eslint.org/docs/latest/use/configure/ignore
- Global ignores also apply to code blocks a processor extracts. https://eslint.org/docs/latest/use/configure/plugins

```js
import { defineConfig, globalIgnores } from "eslint/config";
export default defineConfig([
  globalIgnores([".config/", "dist/"], "project/ignores"),
  { files: ["**/*.js"], ignores: ["**/*.config.js", "!**/eslint.config.js"], rules: { semi: "error" } },
]);
```

https://eslint.org/docs/latest/use/configure/ignore , https://eslint.org/docs/latest/use/configure/configuration-files

## Default ignores and dotfiles

- Default: `["**/node_modules/", ".git/"]`. Your patterns are added to it. https://eslint.org/docs/latest/use/configure/ignore
- Dotfiles and dot directories are linted unless you ignore them. `.git` is the only dot directory ignored by default. Add `"**/.*"` to ignore them all. https://eslint.org/docs/latest/use/configure/ignore , https://eslint.org/docs/latest/use/configure/migration-guide
- Patterns are relative to the config file, or to the cwd with `--config` or `--ignore-pattern`. https://eslint.org/docs/latest/use/configure/ignore
- Unlike `.gitignore`, a pattern like `.config` matches only next to the config file. Use `**/.config/` for any depth, and `**/temp.js` for a file name anywhere. https://eslint.org/docs/latest/use/configure/ignore , https://eslint.org/docs/latest/use/configure/migration-guide

## Pattern recipes

- `build/**` ignores the directory itself, so nothing inside can be unignored. Use `build/**/*`, then negate. https://eslint.org/docs/latest/use/configure/ignore
- Keep only `test.js` under `build/` at any depth: `"build/**/*", "!build/**/*/", "!build/**/test.js"`. A pattern ending in `/` matches only directories. https://eslint.org/docs/latest/use/configure/ignore
- Unignore one package in `node_modules`: `"!node_modules/", "node_modules/*", "!node_modules/mylibrary/"`. https://eslint.org/docs/latest/use/configure/ignore
- Lint only some JS: `globalIgnores(["**/*.js", "**/*.cjs", "**/*.mjs", "!**/src/**/*.js"])`. https://eslint.org/docs/latest/use/configure/ignore

## Importing `.gitignore`

- `includeIgnoreFile()` from `eslint/config` turns an ignore file into global ignores. It accepts one path or an array, plus an optional name. https://eslint.org/docs/latest/use/configure/ignore
- For a `.gitignore`, always pass `gitignoreResolution: true`. The default resolves patterns relative to the ESLint config file. https://eslint.org/docs/latest/use/configure/ignore

```js
import { defineConfig, includeIgnoreFile } from "eslint/config";
// gitignorePath: path to the project's .gitignore
export default defineConfig([includeIgnoreFile(gitignorePath, { gitignoreResolution: true })]);
```

https://eslint.org/docs/latest/use/configure/ignore

## CLI and warnings

- A directory argument skips ignored files silently. An ignored file passed by name warns "File ignored because of a matching ignore pattern". https://eslint.org/docs/latest/use/configure/ignore
- `--no-warn-ignored` silences that warning, which otherwise counts against `--max-warnings 0`. `--no-ignore` disables both config `ignores` and `--ignore-pattern`. https://eslint.org/docs/latest/use/command-line-interface
- Quote `--ignore-pattern` values and repeat the flag: it does not accept comma lists. Unignore with `--ignore-pattern '!node_modules/'`. https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/use/configure/ignore
- A non-JS file that no config object matches is reported as ignored for missing configuration. https://eslint.org/docs/latest/use/configure/configuration-files

## Old patterns

| Old | Current |
| :-- | :-- |
| `.eslintignore`, `ignorePatterns` | Config object with only `ignores`, or `globalIgnores()` |
| `--ignore-path .gitignore` | `includeIgnoreFile(path, { gitignoreResolution: true })` |
| `includeIgnoreFile` from `@eslint/compat` | `includeIgnoreFile` from `eslint/config` (`@eslint/config-helpers`) |
| `temp.js` in `.eslintignore` (any depth) | `**/temp.js` |
| Dotfiles ignored by default | Add `"**/.*"` explicitly |

https://eslint.org/docs/latest/use/configure/migration-guide
