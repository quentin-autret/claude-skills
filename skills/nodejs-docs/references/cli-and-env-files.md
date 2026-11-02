# CLI options, environment variables and .env files

## Contents

- Entry point and module type
- NODE_OPTIONS and config files
- .env files
- node --run and --watch
- Memory and compile cache
- Deprecation and diagnostic flags
- Version-flagged flags
- Old patterns

## Entry point and module type

- The entry is ESM when it is `.mjs`, `.mts` or `.wasm`, when a flag like `--import` forces the ESM loader, or when it is not `.cjs` and the nearest `package.json` has `"type": "module"`. https://nodejs.org/docs/latest/api/cli.html#program-entry-point
- `--input-type=module|commonjs` selects the format of `--eval` and stdin. https://nodejs.org/docs/latest/api/cli.html

## NODE_OPTIONS and config files

- `NODE_OPTIONS` is applied before the command line, so CLI flags win for single-value options and repeatable flags are concatenated. Quote paths with spaces: `NODE_OPTIONS='--require "./my path/file.js"'`. https://nodejs.org/docs/latest/api/cli.html#node_optionsoptions
- `--config-file=path` (renamed from `--experimental-config-file` and stable in v26.12.0) reads `node.config.json` from the cwd by default. Only the `=` form works. Keys: `nodeOptions` (flags allowed in `NODE_OPTIONS`), and the `test`, `watch`, `permission` namespaces. https://nodejs.org/docs/latest/api/cli.html#--config-filepath---config-file
- `NODE_EXTRA_CA_CERTS` is read only at startup, ignored for setuid processes and when `ca` is passed explicitly. A bad file only warns. https://nodejs.org/docs/latest/api/cli.html#node_extra_ca_certsfile
- `FORCE_COLOR` (`1`, `2`, `3`) overrides `NO_COLOR` and `NODE_DISABLE_COLORS`. https://nodejs.org/docs/latest/api/cli.html#force_color1-2-3

## .env files

- Load with `--env-file=.env` (error if missing) or `--env-file-if-exists=.env`. Both are stable since v24.10.0. Real environment variables win over file values, and later files override earlier ones. https://nodejs.org/docs/latest/api/cli.html#--env-filefile

```bash
node --env-file=.env --env-file=.development.env index.js
```

- Values are always strings (`0` stays `'0'`). Quotes are optional, quoted values may span lines, `#` starts a comment outside quotes, and an `export` prefix is ignored. `NODE_OPTIONS` inside the file is applied. https://nodejs.org/docs/latest/api/environment_variables.html#dotenv , https://nodejs.org/docs/latest/api/cli.html#--env-filefile

```text
MY_VAR_A = "# this is NOT a comment"
export MY_VAR = my variable
```

- From code: `process.loadEnvFile(path)` or `util.parseEnv(content)`. https://nodejs.org/docs/latest/api/environment_variables.html#programmatic-apis
- `--env-file` values do not reach scripts run by `node --run`. https://nodejs.org/docs/latest/api/cli.html#--run
- Env files are read before the Permission Model starts and are not restricted by it. https://nodejs.org/docs/latest/api/permissions.html#permission-model-constraints

## node --run and --watch

- `node --run <script>` runs a `package.json` script from that package's directory with every ancestor `node_modules/.bin` on `PATH`. Unlike `npm run` it runs no `pre`/`post` scripts and sets no package-manager variables. Arguments after `--` are appended. https://nodejs.org/docs/latest/api/cli.html#--run

```console
$ node --run test -- --verbose
```

- `--watch` is stable and needs a file (exit code 9 otherwise). `--watch-path` disables watching of imported modules and works only on macOS and Windows. `--run` ignores `--watch`. https://nodejs.org/docs/latest/api/cli.html#--watch

## Memory and compile cache

- `--max-old-space-size` is in MiB. On a 2 GiB machine the docs suggest about 1536. V8 options carry no stability guarantee. https://nodejs.org/docs/latest/api/cli.html#--max-old-space-sizesize-in-mib
- `NODE_COMPILE_CACHE=dir` (stable since v25.4.0) caches compiled code on disk. `NODE_COMPILE_CACHE_PORTABLE=1` survives moving the project, `NODE_COMPILE_CACHE_READONLY=1` (added in v26.8.0) uses a shipped cache, `NODE_DISABLE_COMPILE_CACHE=1` turns it off. https://nodejs.org/docs/latest/api/cli.html#node_compile_cachedir

## Deprecation and diagnostic flags

- Run CI with `--pending-deprecation` (or `NODE_PENDING_DEPRECATION=1`) to surface the documentation-only deprecations labeled as supporting it, and to warn for code loaded from `node_modules`. `--throw-deprecation` makes runtime deprecations throw. https://nodejs.org/docs/latest/api/cli.html#--pending-deprecation , https://nodejs.org/docs/latest/api/deprecations.html
- `--trace-uncaught`, `--trace-exit` and `--abort-on-uncaught-exception` help locate crashes. https://nodejs.org/docs/latest/api/cli.html#--trace-uncaught
- `--process-timeout=30s` (added in v26.11.0, active development) exits with code 124, printing what the main thread did and what kept the loop alive. It skips `'beforeExit'` and `'exit'`, is not allowed in `NODE_OPTIONS` or with inspector flags, and pairs with `--report-on-process-timeout`. https://nodejs.org/docs/latest/api/cli.html#--process-timeoutduration

## Version-flagged flags

| Flag | Added | Stability |
| :-- | :-- | :-- |
| `--allow-net` | v25.0.0 | 1.1 |
| `--permission-audit` | v25.8.0 | |
| `--experimental-stream-iter` | v25.9.0 | 1 |
| `--allow-ffi` | v26.1.0 | 1.1 |
| `--test-randomize`, `--test-random-seed` | v26.1.0 | |
| `--experimental-test-tag-filter` | v26.2.0 | 1.0 |
| `--experimental-vfs`, `--experimental-package-map` | v26.4.0 | 1 |
| `--allow-openssl-store` | v26.7.0 | 1.1 |
| `--allow-fs-vfs`, `--experimental-bench` | v26.9.0 | 1.1, 1 |
| `--process-timeout` | v26.11.0 | 1.1 |

Source: https://nodejs.org/docs/latest/api/cli.html

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `--experimental-config-file` | `--config-file` | https://nodejs.org/docs/latest/api/cli.html#--config-filepath---config-file |
| `--no-experimental-strip-types` | `--no-strip-types` | https://nodejs.org/docs/latest/api/cli.html#--no-strip-types |
| `--no-experimental-require-module` | `--no-require-module` | https://nodejs.org/docs/latest/api/cli.html#--no-experimental-require-module |
| `--watch` combined with `--run` | `--run` ignores it, watch the file directly | https://nodejs.org/docs/latest/api/cli.html#--run |
