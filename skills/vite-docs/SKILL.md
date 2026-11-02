---
name: vite-docs
description: Teaches current Vite 8 practices drawn from the official documentation. Use when writing or fixing a vite.config, migrating from Vite 7 (Rolldown and Oxc replacing Rollup and esbuild), configuring the dev server, proxy or HMR, handling import.meta.env and modes, tuning the production build, library mode, SSR, writing a Vite plugin, using the HMR or JavaScript API, or the Environment API. Triggers on vite.config, defineConfig, rolldownOptions, import.meta.env, loadEnv, optimizeDeps, ssrLoadModule, ModuleRunner, this.environment, upgrade to Vite 8.
metadata:
  version: "Vite 8.x (npm latest 8.3.5)"
  source: https://vite.dev/guide/
  fetched: "2026-11-02"
---

# Vite 8

Vite 8.x, from https://vite.dev/guide/. Each claim ends with its source page.

## How to navigate

1. Read the quick rules below. They catch the most frequent mistakes.
2. Open the one reference file that matches the task, from the table at the end.
3. For `rollupOptions`, `esbuild` or `manualChunks`, read `references/migration.md` first.

## Quick start

```ts
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '') // .env files are not in process.env here
  return {
    plugins: [react()],
    server: { port: Number(env.PORT ?? 5173), strictPort: true },
    build: {
      rolldownOptions: { output: { /* Rolldown options, not Rollup */ } },
    },
  }
})
```

```json
{ "compilerOptions": { "types": ["vite/client"], "isolatedModules": true } }
```

- Node.js 20.19+ or 22.12+. Scaffold with `npm create vite@latest my-app -- --template react-ts`. https://vite.dev/guide/
- Scripts: `vite` (dev, port 5173), `vite build`, `vite preview` (port 4173, local check only, never a production server). https://vite.dev/guide/cli , https://vite.dev/config/preview-options
- Vite never type-checks. Run `tsc --noEmit` next to it. https://vite.dev/guide/features#transpile-only

## Quick rules

### Toolchain and config

- Rolldown bundles builds and pre-bundles dependencies. Oxc transforms and minifies JS, Lightning CSS minifies CSS. https://vite.dev/guide/why#a-unified-toolchain , https://vite.dev/guide/migration#dependency-optimizer-now-uses-rolldown , https://vite.dev/guide/migration#javascript-transforms-by-oxc , https://vite.dev/guide/migration#css-minification-by-lightning-css
- Write `build.rolldownOptions`, `worker.rolldownOptions`, `optimizeDeps.rolldownOptions` and `oxc`. The `rollupOptions`, `esbuildOptions` and `esbuild` keys are deprecated. https://vite.dev/guide/migration#other-related-deprecations , https://vite.dev/guide/migration#javascript-transforms-by-oxc
- Declare entries with the top-level `input` option. It is the default for `build.rolldownOptions.input`, `build.lib.entry` and `optimizeDeps.entries`. https://vite.dev/config/shared-options#input
- Chunking goes through `build.rolldownOptions.output.codeSplitting`, not `manualChunks`. https://vite.dev/guide/build#chunking-strategy
- `tsconfig` `target` is ignored: set `build.target` (default `'baseline-widely-available'`). Vite adds no polyfills, use `@vitejs/plugin-legacy`. https://vite.dev/guide/features#target , https://vite.dev/config/build-options#build-target , https://vite.dev/guide/build#browser-compatibility
- Oxc cannot lower native decorators or mangle properties. Add Babel or SWC filtered to files containing `@`. https://vite.dev/guide/migration#javascript-transforms-by-oxc , https://vite.dev/guide/migration#javascript-minification-by-oxc
- A default import from CommonJS may now return the whole `module.exports` ("got: object", "is not a function"). https://vite.dev/guide/migration#consistent-commonjs-interop , https://vite.dev/guide/troubleshooting#default-import-unexpectedly-returns-an-object

### Env and modes

- Only `VITE_` variables reach client code, always as strings, inlined in the bundle. Never put secrets there, and never set `envPrefix: ''`. https://vite.dev/guide/env-and-mode#env-variables , https://vite.dev/config/shared-options#envprefix
- `.env` files are loaded after the config runs. In `vite.config`, use `loadEnv(mode, root, '')`. https://vite.dev/config/#using-environment-variables-in-config
- Shell variables beat `.env` files. `.env` edits need a restart. https://vite.dev/guide/env-and-mode#env-files
- `--mode` and `NODE_ENV` are independent: `vite build --mode staging` is still a production build. https://vite.dev/guide/env-and-mode#node-env-and-modes
- Write `import.meta.env.BASE_URL` literally, the bracket form is not replaced. https://vite.dev/guide/build#public-base-path

### Dev server

- Never set `server.allowedHosts: true` or `server.cors: true`: any website could read your source. https://vite.dev/config/server-options#server-allowedhosts , https://vite.dev/config/server-options#server-cors
- A busy port silently moves to the next one unless `strictPort: true`. https://vite.dev/config/server-options#server-port
- WebSocket settings live in `server.ws`. `server.hmr` keeps only `boolean | { overlay }`. https://vite.dev/config/server-options#server-hmr
- After `npm link` or editing a linked dependency, restart with `vite --force`. https://vite.dev/guide/troubleshooting#outdated-pre-bundled-deps-when-linking-to-a-local-package
- A full reload instead of HMR can come from a circular dependency (`vite --debug hmr`). https://vite.dev/guide/troubleshooting#a-full-reload-happens-instead-of-hmr

### Build and deploy

- Serve HTML with `Cache-Control: no-cache` and reload on `vite:preloadError`, or old pages break after a deploy. https://vite.dev/guide/build#load-error-handling
- `import.meta.glob` arguments must be literals. A dynamic `import()` must start with `./`, `../` or `#` and end with a file extension: `` import(`./dir/${name}.js`) ``. https://vite.dev/guide/features#glob-import-caveats , https://vite.dev/guide/features#dynamic-import
- CSS as a string: `import css from './a.css?inline'`. Default imports of `.css` are gone. https://vite.dev/guide/features#disabling-css-injection-into-the-page
- Library mode statically replaces `import.meta.env.*` but leaves `process.env.*` untouched. https://vite.dev/guide/build#library-mode

### Plugins, HMR, SSR

- Use hook filters, `transform: { filter: { id: /\.ext$/ }, handler }`, and return `moduleType: 'js'` when converting non-JS. https://vite.dev/guide/api-plugin#hook-filters , https://vite.dev/guide/migration#module-type-support-and-auto-detection
- Read `this.environment` in hooks rather than the `ssr` flag. `hotUpdate` is the environment-aware counterpart of `handleHotUpdate`, whose deprecation is only planned for a future major. https://vite.dev/guide/api-environment-plugins#accessing-the-current-environment-in-hooks , https://vite.dev/changes/hotupdate-hook
- `configureServer` middlewares run before Vite's. Return a function to run after. https://vite.dev/guide/api-plugin#configureserver
- Write `import.meta.hot.accept(` literally inside `if (import.meta.hot)`. https://vite.dev/guide/api-hmr#hot-accept-cb
- SSR dev: `server.middlewareMode: true` plus `appType: 'custom'`, load the entry with `server.environments.ssr.runner.import()` (`ssrLoadModule` is planned for deprecation). https://vite.dev/guide/ssr#setting-up-the-dev-server , https://vite.dev/guide/api-environment-frameworks#runnabledevenvironment , https://vite.dev/changes/ssr-using-modulerunner
- `vite build` builds the client only. `vite build --app` builds every environment. https://vite.dev/guide/api-environment-frameworks#environments-during-build

## Old patterns

| Old | Current |
| :-- | :-- |
| `build.rollupOptions` | `build.rolldownOptions` |
| `esbuild: { jsx..., jsxInject }` | `oxc: { jsx: {...}, jsxInject }` |
| `optimizeDeps.esbuildOptions` | `optimizeDeps.rolldownOptions` |
| `output.manualChunks` | `output.codeSplitting` |
| `server.hmr.port`, `clientPort` | `server.ws.port`, `clientPort` |
| `build.polyfillModulePreload` | `build.modulePreload.polyfill` |
| `import s from './a.css'` | `import s from './a.css?inline'` |
| `import.meta.hot.decline()` (a noop) | `import.meta.hot.invalidate()` |
| `server.ssrLoadModule(url)` | `environment.runner.import(url)` |
| `handleHotUpdate` + `server.ws.send` | `hotUpdate` + `this.environment.hot.send` |

## Reference files

| File | Read it for |
| :-- | :-- |
| [references/migration.md](references/migration.md) | Vite 7 to 8: renamed options, CommonJS interop, removed features |
| [references/config-and-env.md](references/config-and-env.md) | Config, `loadEnv`, `import.meta.env`, modes, `base` |
| [references/code-features.md](references/code-features.md) | TypeScript, JSX, HTML, CSS, assets, `import.meta.glob`, dynamic imports, workers, WebAssembly |
| [references/dev-server.md](references/dev-server.md) | Host, port, `allowedHosts`, CORS, proxy, middleware mode, preview |
| [references/dependencies-and-performance.md](references/dependencies-and-performance.md) | Pre-bundling, `optimizeDeps`, linked packages, dev speed |
| [references/build.md](references/build.md) | Targets, `rolldownOptions`, chunks, library mode |
| [references/backend-integration-and-deploy.md](references/backend-integration-and-deploy.md) | Backend integration, manifest, static hosting |
| [references/ssr.md](references/ssr.md) | SSR dev server, production build, externals, plugins |
| [references/plugins.md](references/plugins.md) | Plugin hooks, filters, virtual modules, ordering |
| [references/hmr-and-javascript-api.md](references/hmr-and-javascript-api.md) | `import.meta.hot`, custom events, `createServer`, `build`, `preview` |
| [references/environment-api.md](references/environment-api.md) | `environments` config, `this.environment`, `configEnvironment`, per-environment state |
| [references/environment-frameworks.md](references/environment-frameworks.md) | `builder.buildApp`, `ModuleRunner` |

Pages read: `SOURCES.md`.
