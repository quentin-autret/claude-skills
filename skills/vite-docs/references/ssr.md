# Server-side rendering

## Contents

- Scope
- Dev server in middleware mode
- Production build
- Preload directives
- SSR externals
- Target, bundling and conditions
- SSR in plugins
- Old patterns

## Scope

- The SSR API is low-level, for framework authors. For an app, start from the higher-level SSR plugins and tools (Awesome Vite SSR section). Reference setups: `npm create vite@latest`, then `Others > create-vite-extra`. https://vite.dev/guide/ssr , https://vite.dev/guide/ssr#example-projects
- For traditional server-side frameworks, use backend integration instead. https://vite.dev/guide/ssr
- Typical files: `index.html` with a placeholder such as `<!--ssr-outlet-->` and a script to `/src/entry-client.js`, `src/entry-server.js` exporting `render(url)`, and `server.js`. https://vite.dev/guide/ssr#source-structure
- Branch with `import.meta.env.SSR`: it is statically replaced, so the other branch is tree-shaken. https://vite.dev/guide/ssr#conditional-logic

## Dev server in middleware mode

Keep your own server in charge and decouple Vite from production. The current recommended form loads the server entry through the environment's module runner: https://vite.dev/guide/api-environment-frameworks#runnabledevenvironment , https://vite.dev/changes/ssr-using-modulerunner

```js
import fs from 'node:fs'
import path from 'node:path'
import express from 'express'
import { createServer, isRunnableDevEnvironment } from 'vite'

const app = express()
const vite = await createServer({ server: { middlewareMode: true }, appType: 'custom' })
app.use(vite.middlewares)

app.use('*all', async (req, res, next) => {
  const url = req.originalUrl
  try {
    let template = fs.readFileSync(path.resolve(import.meta.dirname, 'index.html'), 'utf-8')
    template = await vite.transformIndexHtml(url, template) // HMR client, plugin preambles
    const ssr = vite.environments.ssr
    if (!isRunnableDevEnvironment(ssr)) throw new Error('ssr environment is not runnable')
    const { render } = await ssr.runner.import('/src/entry-server.js')
    const appHtml = await render(url)
    const html = template.replace('<!--ssr-outlet-->', () => appHtml)
    res.status(200).set({ 'Content-Type': 'text/html' }).end(html)
  } catch (e) {
    next(e)
  }
})
```

- `vite.ssrLoadModule(url)` still works and the SSR guide still shows it, with `vite.ssrFixStacktrace(e)` in the `catch`. It is planned for removal in a future major: detect it with `future.removeSsrLoadModule: 'warn'`. https://vite.dev/guide/ssr#setting-up-the-dev-server , https://vite.dev/changes/ssr-using-modulerunner
- `appType: 'custom'` stops Vite from serving HTML itself. https://vite.dev/guide/ssr#setting-up-the-dev-server
- Pass a function to `template.replace` when injecting the rendered HTML, as the guide does. https://vite.dev/guide/ssr#setting-up-the-dev-server
- `vite.middlewares` keeps the same reference across server restarts (config edits), so `app.use` stays valid. https://vite.dev/guide/ssr#setting-up-the-dev-server
- The module runner fixes stack traces itself, so `ssrFixStacktrace` is unnecessary with it (unless `sourcemapInterceptor: false`). https://vite.dev/changes/ssr-using-modulerunner
- In HMR-capable server environments, add `if (import.meta.hot) import.meta.hot.accept()` to the server entry. Without it, a server file change invalidates the whole server module graph. https://vite.dev/guide/api-environment-frameworks#runnabledevenvironment
- A runnable environment runs in the same runtime as the Vite server. For other runtimes (workers, edge), frameworks use a `FetchableDevEnvironment`. https://vite.dev/guide/api-environment-frameworks#devenvironment-communication-levels
- Change the `dev` script from `vite` to `node server`. https://vite.dev/guide/ssr#setting-up-the-dev-server

### Editing a server-only module does not reload the browser

By default, updating a module imported only by the SSR environment does not refresh the page. Frameworks handle it. In a custom setup, add a plugin to the `plugins` passed to `createServer`: https://vite.dev/guide/ssr#setting-up-the-dev-server

```ts
import type { EnvironmentModuleNode, Plugin } from 'vite'

export function ssrReload(): Plugin {
  return {
    name: 'ssr-reload',
    enforce: 'post',
    hotUpdate: {
      order: 'post',
      handler({ modules, server, timestamp }) {
        if (this.environment.name !== 'ssr') return

        const invalidatedModules = new Set<EnvironmentModuleNode>()
        let hasSsrOnlyModules = false

        for (const mod of modules) {
          if (mod.file == null) continue
          const clientModules =
            server.environments.client.moduleGraph.getModulesByFile(mod.file)
          if (clientModules != null) continue

          this.environment.moduleGraph.invalidateModule(
            mod,
            invalidatedModules,
            timestamp,
            true,
          )
          hasSsrOnlyModules = true
        }

        if (hasSsrOnlyModules) {
          server.environments.client.hot.send({ type: 'full-reload' })
          return []
        }
      },
    },
  }
}
```

## Production build

```json
{
  "scripts": {
    "dev": "node server",
    "build:client": "vite build --outDir dist/client --ssrManifest",
    "build:server": "vite build --outDir dist/server --ssr src/entry-server.js"
  }
}
```

- `--ssr` marks the SSR build and names its entry. https://vite.dev/guide/ssr#building-for-production
- In production, `server.js` uses `dist/client/index.html` as the template (it has the hashed asset links), imports `./dist/server/entry-server.js` with plain `import()`, serves `dist/client` statically, and never creates the Vite server. Keep every Vite call behind a dev-only branch. https://vite.dev/guide/ssr#building-for-production
- `vite build` and `vite build --ssr` build one side only. `vite build --app` (or any `builder` config, even `{}`) builds every environment, in `environments` order. https://vite.dev/guide/api-environment-frameworks#environments-during-build
- Non-client builds emit no static assets by default (the client build has them). https://vite.dev/config/build-options#build-emitassets
- SSR builds are not minified by default (`build.minify` is `false`). https://vite.dev/config/build-options#build-minify
- Pre-rendering known routes to HTML (SSG) reuses the same production render. https://vite.dev/guide/ssr#pre-rendering-ssg

## Preload directives

- `--ssrManifest` on the client build writes `dist/client/.vite/ssr-manifest.json`, mapping module ids to client chunks and assets. https://vite.dev/guide/ssr#generating-preload-directives
- The framework collects the module ids used while rendering (with `@vitejs/plugin-vue`, `ctx.modules` after `renderToString(app, ctx)`). Pass the manifest to `render` to emit preload links, or 103 Early Hints. https://vite.dev/guide/ssr#generating-preload-directives

## SSR externals

- Dependencies are externalized by default except linked packages, which go through Vite so HMR works. https://vite.dev/guide/ssr#ssr-externals , https://vite.dev/config/ssr-options#ssr-external
- Add a dependency to `ssr.noExternal` when it ships untranspiled Vite features (CSS imports, `import.meta.glob`). Add a linked package to `ssr.external` to treat it as installed. https://vite.dev/guide/ssr#ssr-externals
- An explicit `ssr.external` array beats `noExternal`. If both are `true`, `noExternal` wins. With `ssr.target: 'node'`, Node built-ins stay external. https://vite.dev/config/ssr-options#ssr-noexternal
- Aliasing one package to another breaks externalized SSR imports: alias at the package manager level (`npm:` alias). https://vite.dev/guide/ssr#ssr-externals

## Target, bundling and conditions

- `ssr.target` defaults to `'node'`. `'webworker'` targets worker runtimes. https://vite.dev/config/ssr-options#ssr-target
- One-file server bundle: `ssr.noExternal: true`. Importing a Node built-in then throws. https://vite.dev/guide/ssr#ssr-bundle
- `ssr.resolve.conditions` default `['module', 'node', 'development|production']` (`browser` instead of `node` for `webworker`). They apply only to bundled deps. Externalized imports use `ssr.resolve.externalConditions` (default `['node', 'module-sync']`). https://vite.dev/config/ssr-options#ssr-resolve-conditions
- Custom external conditions need the same Node flag in dev and production: https://vite.dev/config/ssr-options#ssr-resolve-externalconditions

```sh
NODE_OPTIONS='--conditions custom' vite
NODE_OPTIONS="--conditions custom" node ./dist/server.js
```

- With `vite dev` or `vite preview` serving SSR, add the SSR middleware from `configureServer` / `configurePreviewServer` as a post hook (return a function) so it runs after Vite's middlewares. https://vite.dev/guide/ssr#vite-cli

## SSR in plugins

- Current: branch on the environment in per-environment hooks. https://vite.dev/changes/this-environment-in-hooks#migration-guide

```js
transform(code, id) {
  if (this.environment.config.consumer === 'server') { /* server transform */ }
}
```

- Better: read the precise option using fine-grained environment options rather than the environment name. https://vite.dev/changes/this-environment-in-hooks#migration-guide
- Still supported: `options?.ssr` in `resolveId`, `load`, `transform` (the argument is optional, so use `?.`). Detect it with `future.removePluginHookSsrArgument: 'warn'`. https://vite.dev/guide/ssr#ssr-specific-plugin-logic

## Old patterns

| Old | Current |
| :-- | :-- |
| `await vite.ssrLoadModule('/src/entry-server.js')` | `await vite.environments.ssr.runner.import('/src/entry-server.js')` (guard with `isRunnableDevEnvironment`) |
| `vite.ssrFixStacktrace(e)` | Not needed with the module runner |
| `transform(code, id, ssr)` positional boolean (before Vite 2.7) | `this.environment`, or `options?.ssr` |
| `handleHotUpdate` + `server.ws.send` for SSR reloads | `hotUpdate` + `this.environment.hot.send` (https://vite.dev/changes/hotupdate-hook) |
