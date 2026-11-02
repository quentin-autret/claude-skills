# Environment API: configuration and plugins

## Contents

- Status and who needs it
- Configuring environments
- Plugins and `this.environment`
- Registering and configuring environments from a plugin
- Per-environment state and plugins
- Old patterns

## Status and who needs it

- The Environment API is in release candidate phase. The docs promise stability of the APIs between major releases, and some specific APIs are still considered experimental. https://vite.dev/guide/api-environment , https://vite.dev/changes/
- A plain SPA or MPA needs no environment config: top-level options configure the `client` environment. https://vite.dev/guide/api-environment#closing-the-gap-between-build-and-dev
- The older server APIs are not deprecated. The docs do not recommend migrating plugins and servers yet. Use the `future` flags to find usages. https://vite.dev/guide/api-environment#backward-compatibility , https://vite.dev/changes/

## Configuring environments

```js
export default defineConfig({
  build: { sourcemap: false },          // inherited by every environment
  optimizeDeps: { include: ['lib'] },   // client only, stays top level
  environments: {
    server: {},
    edge: { resolve: { noExternal: true } },
  },
})
```

- Each environment accepts `define`, `resolve`, `optimizeDeps`, `consumer: 'client' | 'server'`, `dev` and `build`, and inherits the top-level values. Configure `client` through top-level options, so adding environments does not change it. https://vite.dev/guide/api-environment#environments-configuration
- In dev, `client` and an environment named `ssr` always exist. In build, `client` always exists and `ssr` only when configured (`environments.ssr` or `build.ssr`). The server environment can have any name. https://vite.dev/guide/api-environment#environments-configuration
- The top-level `ssr` option will be deprecated once the API is stable. https://vite.dev/guide/api-environment#environments-configuration
- All environments share one dev server: HTTP server, middlewares, resolved config, plugin pipeline. The browser runs `client` code, a module runner runs the others. https://vite.dev/guide/api-environment#closing-the-gap-between-build-and-dev

## Plugins and `this.environment`

- Global hooks (`config`, `configResolved`, `configureServer`...) run once. Per-environment hooks (all Rolldown hooks, `hotUpdate`, other Vite-specific module hooks) run per environment with `this.environment`. https://vite.dev/guide/api-environment-plugins#per-environment-hooks-and-global-hooks
- Read the environment from the hook context instead of caching the server from `configureServer`: https://vite.dev/guide/api-environment-plugins#accessing-the-current-environment-in-hooks , https://vite.dev/changes/this-environment-in-hooks#motivation

```ts
transform(code, id) {
  console.log(this.environment.name, this.environment.config.resolve.conditions)
}
```

- Build hooks (`renderChunk`, `generateBundle`...) also get `this.environment`. https://vite.dev/guide/api-environment-plugins#environment-in-build-hooks
- Prefer checking a precise option over the environment name. `consumer === 'server'` is the quick replacement for `options.ssr`. https://vite.dev/changes/this-environment-in-hooks#migration-guide
- `buildStart`, `buildEnd` and `watchChange` run for `client` only in dev, unless `perEnvironmentStartEndDuringDev: true` / `perEnvironmentWatchChangeDuringDev: true`. https://vite.dev/guide/api-environment-plugins#per-environment-state-in-plugins
- `hotUpdate` runs once per environment, in `server.environments` order, with that environment's modules. It also fires on `create` and `delete`, unlike `handleHotUpdate`. https://vite.dev/guide/api-environment-plugins#the-hotupdate-hook , https://vite.dev/changes/hotupdate-hook#motivation
- Each environment has its own module graph. The mixed Vite 5 graph (`clientImportedModules`, `ssrImportedModules`) only survives as the `server.moduleGraph` compatibility view. https://vite.dev/guide/api-environment-instances#separate-module-graphs

## Registering and configuring environments from a plugin

Register in `config` (an empty object is enough). Configure each environment in `configEnvironment`, because the full list is unknown during `config`: https://vite.dev/guide/api-environment-plugins#registering-new-environments-using-hooks , https://vite.dev/guide/api-environment-plugins#configuring-environment-using-the-configenvironment-hook

```ts
const plugin = {
  name: 'rsc',
  config() {
    return { environments: { rsc: { resolve: { conditions: ['react-server', ...defaultServerConditions] } } } }
  },
  configEnvironment(name, options) {
    if (name === 'rsc') return { resolve: { conditions: ['workerd'] } }
  },
}
```

## Per-environment state and plugins

- One plugin instance serves all environments: key state by environment. `perEnvironmentState` is the helper for it. https://vite.dev/guide/api-environment-plugins#per-environment-state-in-plugins , https://vite.dev/changes/shared-plugins-during-build

```ts
const state = perEnvironmentState<{ count: number }>(() => ({ count: 0 }))
const plugin = {
  name: 'counter',
  buildStart() { state(this).count = 0 },
  transform() { state(this).count++ },
}
```

- `applyToEnvironment(env)` returns `true`, `false` or a replacement plugin. Without it, the plugin is active everywhere. It runs after `configResolved`. https://vite.dev/guide/api-environment-plugins#per-environment-plugins-using-the-applytoenvironment-hook
- Wrap a plugin that is not environment-aware with `perEnvironmentPlugin`. The returned plugin must not use Vite-specific hooks. https://vite.dev/guide/api-environment-plugins#per-environment-plugins-using-the-applytoenvironment-hook

```js
perEnvironmentPlugin('per-env', (environment) => nonShareablePlugin({ outputName: environment.name }))
```

- Talk to the app through `environment.hot`. `vite:client:connect` / `vite:client:disconnect` give a stable `client` per connection. `client.send` targets one instance, `hot.send` broadcasts. https://vite.dev/guide/api-environment-plugins#application-plugin-communication

```js
configureServer(server) {
  server.environments.ssr.hot.on('my:greetings', (data, client) => client.send('my:reply', `You said ${data}`))
}
```

## Old patterns

These still work. Detect them with `future.<flag>: 'warn'` before the next major. https://vite.dev/changes/per-environment-apis#migration-guide , https://vite.dev/changes/ssr-using-modulerunner

| Old | Current | `future` flag |
| :-- | :-- | :-- |
| `options.ssr` in hooks | `this.environment.config.consumer === 'server'` | `removePluginHookSsrArgument` |
| `handleHotUpdate` | `hotUpdate` | `removePluginHookHandleHotUpdate` |
| `server.moduleGraph.getModuleByUrl(url, { ssr })` | `environment.moduleGraph.getModuleByUrl(url)` | `removeServerModuleGraph` |
| `server.reloadModule(m)` | `environment.reloadModule(m)` | `removeServerReloadModule` |
| `server.pluginContainer` | `environment.pluginContainer` | `removeServerPluginContainer` |
| `server.transformRequest(url, ssr)` | `environment.transformRequest(url)` | `removeServerTransformRequest` |
| `server.warmupRequest(url, ssr)` | `environment.warmupRequest(url)` | `removeServerWarmupRequest` |
| `server.hot` | `server.client.environment.hot` | `removeServerHot` |
| `server.ssrLoadModule(url)` | `runner.import(url)` | `removeSsrLoadModule` |
| `build()` for a whole app | `createBuilder()` + `buildApp()` (https://vite.dev/guide/api-environment-frameworks) | none |
