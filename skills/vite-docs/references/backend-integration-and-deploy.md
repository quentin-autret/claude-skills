# Backend integration and static deployment

## Contents

- Backend integration and the manifest
- Static deployment

## Backend integration and the manifest

For a backend (Rails, Laravel, Django) that renders HTML. Look for an existing integration first. https://vite.dev/guide/backend-integration

```js
export default defineConfig({
  input: '/path/to/main.js',
  server: { cors: { origin: 'http://my-backend.example.com' } },
  build: { manifest: true }, // writes outDir/.vite/manifest.json
})
```

- Dev: the backend template loads `http://localhost:5173/@vite/client` and the entry as `<script type="module">`. For assets, either proxy asset requests to Vite or set `server.origin`. https://vite.dev/guide/backend-integration
- With `@vitejs/plugin-react`, add the React Refresh preamble before those scripts, since the plugin cannot edit HTML it does not serve: https://vite.dev/guide/backend-integration

```html
<script type="module">
  import RefreshRuntime from 'http://localhost:5173/@react-refresh'
  RefreshRuntime.injectIntoGlobalHook(window)
  window.$RefreshReg$ = () => {}
  window.$RefreshSig$ = () => (type) => type
  window.__vite_plugin_react_preamble_installed__ = true
</script>
```

- Manifest keys: entries and dynamic entries use the source path relative to the root (`isEntry`, `isDynamicEntry`). Shared chunks use `_` plus the file name. With `cssCodeSplit: false`, CSS is under `style.css`. Types `Manifest` and `ManifestChunk` come from `vite`. https://vite.dev/guide/backend-integration
- Tags for an entry, in order: a stylesheet link per file in its `css`, then stylesheets of every chunk reached through `imports` (recursively), then the entry `file` as `<script type="module">`, then optional `modulepreload` links for imported JS chunks. Vite ships no helper for this walk. https://vite.dev/guide/backend-integration
- With `build.chunkImportMap`, inject `importmap.json` as `<script type="importmap">` before any module script or modulepreload. https://vite.dev/guide/backend-integration

## Static deployment

- `vite preview` is not meant as a production server. Opening `dist/index.html` through `file://` fails with CORS: serve it over HTTP. https://vite.dev/guide/static-deploy#testing-the-app-locally , https://vite.dev/guide/troubleshooting#built-file-does-not-work-because-of-cors-error
- `ENOENT` or `Module not found` errors appear when a project developed on a case-insensitive filesystem (Windows or macOS) is built on a case-sensitive one (Linux): fix the import casing. https://vite.dev/guide/troubleshooting#no-such-file-or-directory-error-due-to-case-sensitivity
- GitHub Pages under `<user>.github.io/<repo>/` needs `base: '/<repo>/'` and the Pages source set to "GitHub Actions". https://vite.dev/guide/static-deploy#github-pages
- Firebase hosting example: a rewrite of `**` to `/index.html` in `firebase.json`. https://vite.dev/guide/static-deploy#google-firebase
- Cloudflare Workers: `@cloudflare/vite-plugin` runs server code in the Workers runtime in dev through the Environment API, then `npx wrangler deploy`. https://vite.dev/guide/static-deploy#cloudflare-workers
