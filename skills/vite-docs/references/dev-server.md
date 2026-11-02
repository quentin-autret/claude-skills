# Dev server and preview server

## Contents

- Host and port
- Security: allowedHosts, CORS, fs
- Proxy
- WebSocket and HMR connection
- File watching
- Middleware mode and appType
- Other server options
- Preview server
- Dev server troubleshooting

## Host and port

- `server.host` defaults to `'localhost'`. `true` or `'0.0.0.0'` listens on every address, LAN and public included. https://vite.dev/config/server-options#server-host
- With `localhost`, Node DNS ordering can make the browser hit a different address. Vite prints the resolved address when it differs. https://vite.dev/config/server-options#server-host
- `server.port` defaults to `5173`. When it is taken, Vite automatically tries the next available port. Set `server.strictPort: true` (CLI `--strictPort`) to exit instead, for example when scripts or proxies expect a fixed port. https://vite.dev/config/server-options#server-port
- VS Code dev containers: set `server.host: '127.0.0.1'`, port forwarding does not support IPv6. https://vite.dev/guide/troubleshooting#dev-containers-vs-code-port-forwarding
- WSL2: `host: true` is not enough for LAN access (WSL networking). https://vite.dev/config/server-options#server-host

## Security: allowedHosts, CORS, fs

- `server.allowedHosts` defaults to `[]`: `localhost`, `*.localhost` and IP addresses are allowed. A leading dot allows subdomains: `['.example.com']`. The check is skipped over HTTPS. https://vite.dev/config/server-options#server-allowedhosts
- Never set `allowedHosts: true`: any website can then reach the dev server through DNS rebinding and download your source. Never list a TLD. List only hosts whose DNS you control. Extra hosts can come from `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` (comma-separated). https://vite.dev/config/server-options#server-allowedhosts
- `server.cors` default allows `localhost`, `127.0.0.1` and `::1` origins. `cors: true` lets any website read your source. Use an explicit origin: https://vite.dev/config/server-options#server-cors

```js
server: { cors: { origin: 'http://my-backend.example.com' } }
```

- `server.fs.strict` (default `true`) returns 403 for files outside `server.fs.allow`. `allow` defaults to the detected workspace root (`workspaces`, `lerna.json`, `pnpm-workspace.yaml`), else the project root. Setting `allow` disables detection, so add it back: https://vite.dev/config/server-options#server-fs-allow

```js
import { defineConfig, searchForWorkspaceRoot } from 'vite'
export default defineConfig({
  server: { fs: { allow: [searchForWorkspaceRoot(process.cwd()), '/path/to/shared'] } },
})
```

- `server.fs.deny` default: `.env`, `.env.*`, certificate and key files, `.npmrc`, `.yarnrc.yml`, `**/.git/**`. It outranks `allow` but does not cover `publicDir`, which is served unfiltered. A file reachable by another path (symlink, plugin `load`) may still leak: deny those paths too. https://vite.dev/config/server-options#server-fs-deny
- `server.https`: `@vitejs/plugin-basic-ssl` gives a cached self-signed certificate for basic setups. Your own trusted certificate is recommended, since Chrome ignores caching directives with a self-signed certificate and requests stop loading. https://vite.dev/config/server-options#server-https , https://vite.dev/guide/troubleshooting#network-requests-stop-loading

## Proxy

```js
server: {
  proxy: {
    '/foo': 'http://localhost:4567',
    '/api': {
      target: 'http://api.example.com',
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api/, ''),
    },
    '^/fallback/.*': { target: 'http://api.example.com', changeOrigin: true },
    '/socket.io': { target: 'ws://localhost:5174', ws: true },
  },
}
```

- Keys starting with `^` are RegExp. Proxied responses are not transformed by Vite. With a non-relative `base`, prefix every key with it. https://vite.dev/config/server-options#server-proxy
- The proxy is built on `http-proxy-3`, and `configure(proxy, options)` receives that instance. https://vite.dev/config/server-options#server-proxy
- Vite does not check the `Origin` of proxied WebSocket requests: the target must. `rewriteWsOrigin: true` defeats that check and opens the proxy to CSRF, so leave it off unless the target is trusted. https://vite.dev/config/server-options#server-proxy
- For custom middleware, write a plugin with `configureServer` instead of abusing the proxy. https://vite.dev/config/server-options#server-proxy

## WebSocket and HMR connection

- WebSocket settings live in `server.ws` (`protocol`, `host`, `port`, `path`, `clientPort`, `timeout` default 30000 ms, `server`). `server.ws: false` disables the socket. The same keys under `server.hmr` are deprecated and synced. https://vite.dev/config/server-options#server-ws , https://vite.dev/config/server-options#server-hmr
- `server.hmr` now takes `boolean | { overlay?: boolean }`. `overlay: false` hides the error overlay. https://vite.dev/config/server-options#server-hmr
- Behind a reverse proxy, the client first tries the proxied socket and falls back to a direct connection (the printed error is harmless). To avoid it, forward WebSocket in the proxy, or set `strictPort: true` with `server.ws.clientPort` equal to `server.port`, or give `server.ws.port` its own port. https://vite.dev/config/server-options#server-ws
- Middleware mode with a proxied WebSocket: pass the parent HTTP server. https://vite.dev/guide/api-javascript#createserver

```js
const parentServer = http.createServer()
const vite = await createServer({
  server: {
    middlewareMode: { server: parentServer },
    proxy: { '/ws': { target: 'ws://localhost:3000', ws: true } },
  },
})
```

## File watching

- Vite watches `root` except `.git/`, `node_modules/`, `test-results/`, `cacheDir` and `build.outDir`. Files inside `node_modules` cannot be watched. `server.watch: null` watches nothing. https://vite.dev/config/server-options#server-watch
- WSL2 and Docker on WSL2 miss edits made by Windows apps. Keep the project in the Linux filesystem and edit from WSL. Escape hatch: `server.watch: { usePolling: true }` (high CPU). The same polling fixes `EISDIR` on VirtualBox shared folders. https://vite.dev/config/server-options#server-watch , https://vite.dev/guide/troubleshooting#illegal-operation-error-when-watching-files
- `ENOSPC: System limit for number of file watchers reached`: exclude heavy folders with `server.watch.ignored`, or raise `fs.inotify.max_user_watches`. https://vite.dev/guide/troubleshooting#vite-crashes-with-enospc-error

## Middleware mode and appType

- `appType` (default `'spa'`) adds HTML middlewares and SPA fallback. `'mpa'` keeps HTML middlewares without fallback. `'custom'` adds none, you serve HTML yourself. https://vite.dev/config/shared-options#apptype
- Embed Vite in your own server: https://vite.dev/config/server-options#server-middlewaremode

```js
const vite = await createServer({ server: { middlewareMode: true }, appType: 'custom' })
app.use(vite.middlewares)
```

- With `'spa'` or `'mpa'` in middleware mode, add your own middlewares before Vite's, since Vite's HTML and 404 handlers end the request. https://vite.dev/config/server-options#server-middlewaremode
- `server.httpServer` and `server.resolvedUrls` are `null` in middleware mode. https://vite.dev/guide/api-javascript#vitedevserver
- `server.origin` sets the origin of generated asset URLs in dev (backend integration). https://vite.dev/config/server-options#server-origin

## Other server options

- `server.warmup.clientFiles` / `ssrFiles` pre-transform hot files to avoid request waterfalls. List only frequently used files. https://vite.dev/config/server-options#server-warmup
- `server.forwardConsole` sends browser errors and `console.warn`/`console.error` to the terminal. It is on automatically when an AI coding agent is detected, off otherwise. Tune it with `{ unhandledErrors: true, logLevels: ['warn', 'error'] }`. https://vite.dev/config/server-options#server-forwardconsole
- `server.open` takes a path. The browser comes from `BROWSER` and `BROWSER_ARGS`. https://vite.dev/config/server-options#server-open
- `server.sourcemapIgnoreList` is independent of the build one and receives absolute paths. Default ignores `node_modules`. https://vite.dev/config/server-options#server-sourcemapignorelist
- `devtools` (experimental) needs `@vitejs/devtools`. Set it in the user config, plugin `config` hooks cannot change it. https://vite.dev/config/shared-options#devtools

## Preview server

- `vite preview` serves the built `dist` on port 4173 to check a build locally. It is not a production server. Run `vite build` first. https://vite.dev/guide/cli , https://vite.dev/guide/static-deploy#testing-the-app-locally
- `preview.host`, `allowedHosts`, `strictPort`, `https`, `open`, `proxy` and `cors` fall back to the `server.*` values. `preview.headers` does not. https://vite.dev/config/preview-options
- A plugin adds preview middleware with `configurePreviewServer`. https://vite.dev/guide/api-plugin#configurepreviewserver

## Dev server troubleshooting

- Requests stalled forever on Linux: raise the file descriptor limit (`ulimit -Sn`) and the inotify limits, then restart the browser. https://vite.dev/guide/troubleshooting#requests-are-stalled-forever
- `431 Request Header Fields Too Large`: shorten cookies or start Node with `--max-http-header-size`. https://vite.dev/guide/troubleshooting#_431-request-header-fields-too-large
- Windows: a `&` in the project path breaks the npm bin shim (`Cannot find module ...vite.js`). Remove it or use pnpm or Yarn. https://vite.dev/guide/troubleshooting#cli
- Windows cross-drive links (`subst`, junctions to another drive) can break Vite. https://vite.dev/guide/troubleshooting#cross-drive-links-on-windows
- White screen with no error, or `Failed to fetch dynamically imported module` in dev: a browser extension (such as an ad blocker) may be blocking the Vite client's requests. Try disabling extensions. https://vite.dev/guide/troubleshooting#browser-extensions
