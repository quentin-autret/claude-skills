# CLI, exit codes, cache, bulk suppressions, flags, MCP server

## Contents

- File arguments and globs
- Fixing from the CLI
- Exit codes, warnings, `--quiet`
- Formatters and output
- Cache
- Bulk suppressions
- Feature flags and other options
- MCP server
- Old patterns

## File arguments and globs

- No file argument means `.`. `--pass-on-no-patterns` exits 0 instead. https://eslint.org/docs/latest/use/command-line-interface
- Quote globs so ESLint expands them, not the shell. Use double quotes on Windows. https://eslint.org/docs/latest/use/command-line-interface
- `--no-error-on-unmatched-pattern` covers unmatched quoted globs only, not a shell glob that matched nothing. https://eslint.org/docs/latest/use/command-line-interface
- Lint extra extensions by adding them to `files` in the config. `--ext` is meant for runs with `--no-config-lookup`. https://eslint.org/docs/latest/use/command-line-interface
- Repeat multi-value options or use commas (`--global describe,it`). `--ignore-pattern` takes no comma list. https://eslint.org/docs/latest/use/command-line-interface
- `--rule` merges with the config. Combine it with `--no-config-lookup` to run only CLI rules. Plugin rules need the `plugin/` prefix, and `--plugin` needs the plugin installed. https://eslint.org/docs/latest/use/command-line-interface
- `--global name` is read-only for `no-undef`. Append `:true` to make it writable. https://eslint.org/docs/latest/use/command-line-interface
- Pass `--stdin-filename` with `--stdin` for rules that depend on the file name. https://eslint.org/docs/latest/use/command-line-interface

```sh
eslint --ext .ts --no-config-lookup .
eslint --rule 'quotes: [error, double]' --no-config-lookup
eslint --ignore-pattern "/lib/" --ignore-pattern "/src/vendor/*" .
```

https://eslint.org/docs/latest/use/command-line-interface

## Fixing from the CLI

- `--fix` writes files and prints only what remains. It throws on piped input. https://eslint.org/docs/latest/use/command-line-interface
- `--fix` does nothing on files handled by a processor unless the processor supports autofix. https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/extend/custom-processors
- `--fix-dry-run` computes fixes without writing. The default formatter does not print them: use `--format json`. https://eslint.org/docs/latest/use/command-line-interface
- `--fix-type problem,suggestion,layout,directive` limits fixes, useful when a formatter owns layout. https://eslint.org/docs/latest/use/command-line-interface

```sh
eslint --stdin --fix-dry-run --format json
eslint --fix --fix-type suggestion,layout .
```

https://eslint.org/docs/latest/use/command-line-interface

## Exit codes, warnings, `--quiet`

| Code | Meaning |
| :-- | :-- |
| 0 | No error, warnings within `--max-warnings` |
| 1 | At least 1 error, or more warnings than `--max-warnings` |
| 2 | Configuration problem or internal error |

https://eslint.org/docs/latest/use/command-line-interface

- `--max-warnings` defaults to -1 (off). https://eslint.org/docs/latest/use/command-line-interface
- `--quiet` reports errors only and does not even run `"warn"` rules. A warn rule that marks variables as used for other rules then has no effect. With `--max-warnings`, warn rules still run but are not reported. https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Parse errors are reported as problems. `--exit-on-fatal-error` exits 2 on them. https://eslint.org/docs/latest/use/command-line-interface

## Formatters and output

- Built-in formatters: `stylish` (default), `json`, `json-with-metadata` (adds rule metadata), `html`. https://eslint.org/docs/latest/use/formatters/
- `-f` takes a built-in name, a local path or a package, with or without the `eslint-formatter-` prefix. A local path must start with `.` (`-f ./customformat.js`). https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/extend/custom-formatters
- `-o file` writes the output, even when empty. https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Colors: `--color` and `--no-color` win over `FORCE_COLOR`, `NO_COLOR` and `NODE_DISABLE_COLORS`. Lost colors in CI can come from those variables. https://eslint.org/docs/latest/use/migrate-to-10.0.0
- `--stats --format json` adds parse, fix and per-rule timings. The environment variable `TIMING=1` (or `TIMING=all`) profiles rules, and in a simple run the sum of per-rule `--stats` times is comparable to its first column (use `TIMING=all`). https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/extend/custom-rules , https://eslint.org/docs/latest/extend/stats

## Cache

- `--cache` stores results in `.eslintcache` and relints changed files only. A later run without `--cache` deletes the file. https://eslint.org/docs/latest/use/command-line-interface
- The cache sees only the file and the config, not dependencies between files. Type-aware rules (typescript-eslint) and import rules (eslint-plugin-import, eslint-plugin-import-x) can return stale results: run without `--cache` when they must be exact. https://eslint.org/docs/latest/use/command-line-interface
- `--cache-strategy content` after a `git clone` or anywhere mtimes change without content changes. Default `metadata`. https://eslint.org/docs/latest/use/command-line-interface
- `--cache-location` for a directory that does not exist yet needs a trailing `/`, otherwise it is read as a file name. https://eslint.org/docs/latest/use/command-line-interface
- ESLint does not clear the cache when plugins are upgraded: delete it yourself. https://eslint.org/docs/latest/integrate/nodejs-api

```sh
eslint "src/**/*.js" --cache --cache-strategy content
```

https://eslint.org/docs/latest/use/command-line-interface

## Bulk suppressions

- Turn on a new rule as `"error"` while existing violations are recorded in `eslint-suppressions.json`. New violations still fail. Only `"error"` rules are suppressed. https://eslint.org/docs/latest/use/suppressions
- Run `--fix` with `--suppress-all` so fixable violations are fixed, not suppressed. Commit `eslint-suppressions.json`. https://eslint.org/docs/latest/use/suppressions
- A custom `--suppressions-location` must be passed on every run, not only when creating. https://eslint.org/docs/latest/use/suppressions
- Fixing a suppressed violation makes ESLint exit 2 ("There are suppressions left that do not occur anymore"). Run `--prune-suppressions`, or pass `--pass-on-unpruned-suppressions`. https://eslint.org/docs/latest/use/suppressions

```sh
eslint --fix --suppress-all
eslint --fix --suppress-rule no-unused-expressions
eslint --prune-suppressions
```

https://eslint.org/docs/latest/use/suppressions

## Feature flags and other options

- `unstable_*` flags are experimental. `v##_*` flags enable the next major's behavior early and throw after that major. https://eslint.org/docs/latest/pages/flags
- Enable with `--flag name`, `ESLINT_FLAGS="a,b"` (merged with other sources, good for CI), or `flags: [...]` in the Node.js API. https://eslint.org/docs/latest/pages/flags
- `--concurrency <n|auto|off>` lints in worker threads. Default `off`. https://eslint.org/docs/latest/use/command-line-interface
- `--init`, `--inspect-config`, `--print-config <file>`, `--env-info` do no linting. Attach `--debug` and `--env-info` output to bug reports. https://eslint.org/docs/latest/use/command-line-interface

## MCP server

- `--mcp` starts the ESLint MCP server. Editors register it for example with `"command": "npx"` and `"args": ["@eslint/mcp@latest"]`. https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/use/mcp
- With a TypeScript config, `npx @eslint/mcp@latest` runs in a temp directory that cannot resolve `jiti`. Install `@eslint/mcp` and `jiti` as dev dependencies and use args `["@eslint/mcp"]`, or on Node >= 22.13.0 set `ESLINT_FLAGS=unstable_native_nodejs_ts_config` and `NODE_OPTIONS=--experimental-transform-types`. https://eslint.org/docs/latest/use/mcp
- Yarn Plug'n'Play is not supported: use `nodeLinker: node-modules`. https://eslint.org/docs/latest/use/mcp

## Old patterns

| Old | Current |
| :-- | :-- |
| `--cache-file` | `--cache-location` |
| `--no-eslintrc` | `--no-config-lookup` |
| `--env` | `languageOptions.globals` in the config |
| `--rulesdir` | Inline plugin in the config |
| `--ignore-path` | `includeIgnoreFile()` in the config |
| `--resolve-plugins-relative-to` | Not needed, plugins are imported |
| `-f checkstyle`, `compact`, `jslint-xml`, `junit`, `tap`, `unix`, `visualstudio` | Install `eslint-formatter-<name>` |
| `--flag v10_config_lookup_from_file` | Default behavior, the flag throws |

https://eslint.org/docs/latest/use/command-line-interface , https://eslint.org/docs/latest/use/configure/migration-guide , https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/use/migrate-to-10.0.0
