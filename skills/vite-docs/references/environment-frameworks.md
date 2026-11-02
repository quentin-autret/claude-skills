# Environment API: frameworks, builds and runtimes

For framework and runtime authors. The API is in release candidate phase, some parts are experimental. https://vite.dev/guide/api-environment

## Contents

- Dev communication levels
- Building all environments
- Module runner and custom runtimes

## Dev communication levels

- `RunnableDevEnvironment`: in-process, passes any JS value, but the runner must use the same runtime as the Vite server. Default for `ssr` and other non-client environments in dev. https://vite.dev/guide/api-environment-frameworks#devenvironment-communication-levels
- `FetchableDevEnvironment`: exchanges Fetch `Request` / `Response`. The docs recommend it over runnable, which works in few runtimes. Vite throws `TypeError` for non-global `Request` or `Response` objects. https://vite.dev/guide/api-environment-frameworks#fetchabledevenvironment
- Raw `DevEnvironment`: pass data through a virtual module when the code runs in the user runtime, or `hot.send` / `import.meta.hot.on` when it needs Node APIs. https://vite.dev/guide/api-environment-frameworks#raw-devenvironment

```js
import { isRunnableDevEnvironment } from 'vite'

const env = server.environments.ssr
if (isRunnableDevEnvironment(env)) {
  const { render } = await env.runner.import('/src/entry-server.js')
}
```

- The runner is created lazily, on first access, and then enables source map support for stack traces. https://vite.dev/guide/api-environment-frameworks#runnabledevenvironment
- Add `if (import.meta.hot) import.meta.hot.accept()` in the server entry, or each server change invalidates the whole server graph. https://vite.dev/guide/api-environment-frameworks#runnabledevenvironment
- Prefer `this.environment` in hooks over reaching into `server.environments`. https://vite.dev/guide/api-environment-frameworks#environment-agnostic-code

## Building all environments

- `vite build` builds `client` only and `vite build --ssr` the SSR side only. Any `builder` config (even `{}`) or `vite build --app` builds every environment, one after another in `environments` order. https://vite.dev/guide/api-environment-frameworks#environments-during-build
- Parallel builds: https://vite.dev/guide/api-environment-frameworks#configuring-the-app-build-with-builder-buildapp

```js
export default defineConfig({
  builder: {
    buildApp: async (builder) => {
      await Promise.all(Object.values(builder.environments).map((env) => builder.build(env)))
    },
  },
})
```

- The `buildApp` plugin hook runs hooks with order `'pre'` or `null`, then `builder.buildApp`, then `post` hooks. Check `environment.isBuilt` to avoid building twice. https://vite.dev/guide/api-environment-frameworks#the-buildapp-plugin-hook
- All environments build in one process. By default each gets its own `ResolvedConfig`, so plugins keep reading `config.build`. `builder.sharedConfigBuild: true` shares one config. A plugin opts into sharing with `sharedDuringBuild: true`. https://vite.dev/guide/api-environment-plugins#shared-plugins-during-build , https://vite.dev/changes/shared-plugins-during-build

## Module runner and custom runtimes

- `ModuleRunner` comes from `vite/module-runner` and is decoupled from the server. Methods: `import`, `clearCache`, `close` (keeps the HMR connection), `isClosed`. https://vite.dev/guide/api-environment-runtimes#modulerunner
- On `full-reload`, the runner re-executes modules and replaces the `exports` object. Code that holds old exports must call `import` again. https://vite.dev/guide/api-environment-runtimes#modulerunner
- `ESModulesEvaluator` uses `new AsyncFunction`. In runtimes that forbid it, write a `ModuleEvaluator`. Inline source maps then need a 2-line offset that only `ESModulesEvaluator` handles. https://vite.dev/guide/api-environment-runtimes#moduleevaluator
- `ModuleRunnerTransport`: provide `invoke`, or `send` plus `connect`. An `invoke`-only (HTTP) transport needs `hmr: false`. Server side: `environment.hot.handleInvoke(payload)`. https://vite.dev/guide/api-environment-runtimes#modulerunnertransport
- `HotChannel` transports get `server.fs` checks. Set `skipFsCheck: true` only when the transport is not reachable over the network. https://vite.dev/guide/api-environment-runtimes#creating-a-new-environment-factory
- Environment factories (for runtime providers) return `EnvironmentOptions` with `dev.createEnvironment` and `build.createEnvironment`, merged with user options. https://vite.dev/guide/api-environment-runtimes#environment-factories

```js
const runner = new ModuleRunner(
  { transport, createImportMeta: createNodeImportMeta },
  new ESModulesEvaluator(),
)
await runner.import('/src/entry-point.js')
```

- `transformRequest` and `warmupRequest` are the current names in the proposal, and may be renamed before release. https://vite.dev/guide/api-environment-instances#devenvironment-class
