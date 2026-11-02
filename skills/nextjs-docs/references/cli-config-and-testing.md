# CLI, next.config, TypeScript, linting and testing

`next dev`, `next build`, `next typegen`, `next analyze`, `next.config` basics, TypeScript integration, ESLint, React Compiler, debugging, memory, Fast Refresh and test runners in Next.js 16.4.4.

## Contents

- `next dev` and `next build`
- Reading and debugging the build
- `next.config` basics
- TypeScript
- ESLint
- React Compiler
- Dev speed, memory and Fast Refresh
- Testing

## `next dev` and `next build`

- `next dev` writes to `.next/dev`, so dev and build can run together. A lockfile blocks a second `next dev` (or `next build`) on the same project.
  https://nextjs.org/docs/app/guides/upgrading/version-16#concurrent-dev-and-build
- `next build` no longer lints and no longer prints size or First Load JS. `next analyze` analyzes bundles.
  https://nextjs.org/docs/app/guides/upgrading/version-16#performance-improvements
  https://nextjs.org/docs/app/guides/upgrading/version-16#next-lint-command
  https://nextjs.org/docs/app/api-reference/cli/next#next-analyze-options
- `PORT` cannot live in `.env` (the server boots before env loading). Use `next dev -p 4000`.
  https://nextjs.org/docs/app/api-reference/cli/next#changing-the-default-port
- Behind a load balancer, set `next start --keepAliveTimeout` (milliseconds, for example 70000) larger than the proxy's timeouts.
  https://nextjs.org/docs/app/api-reference/cli/next#configuring-a-timeout-for-downstream-proxies
- Persist `.next/cache` between CI builds, keyed on the lockfile and source hashes.
  https://nextjs.org/docs/app/guides/ci-build-caching
- `distDir` belongs to Next.js, which may delete its contents. Use a dedicated folder.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/distDir

## Reading and debugging the build

- With Cache Components the route table shows `○` Static, `◐` Partial Prerender, `ƒ` Dynamic.
  https://nextjs.org/docs/app/guides/building#reading-the-build-output
- With Cache Components the build fails when `params`, `cookies()`, `headers()`, `searchParams`, `connection()` or an uncached `fetch` run outside `<Suspense>`. Fixes: stream (Suspense), cache (`use cache`) or allow blocking (`instant = false`).
  https://nextjs.org/docs/app/guides/building#find-blocking-work
- `next build --debug-prerender` keeps server code readable, adds source maps and continues past the first failing route. Never deploy that build.
  ```bash
  next build --debug-prerender --debug-build-paths="app/products/[id]/page.tsx"
  ```
  https://nextjs.org/docs/app/guides/building#debug-prerender-errors

## `next.config` basics

- Use `next.config.ts` with `import type { NextConfig } from 'next'`, or `.mjs` for ESM. `.cjs` and `.cts` are not supported.
  https://nextjs.org/docs/app/api-reference/config/next-config-js#ecmascript-modules
- During `next dev` the config no longer sees `'dev'` in `process.argv`. Check `process.env.NODE_ENV` or the `phase` argument.
  https://nextjs.org/docs/app/guides/upgrading/version-16#next-dev-config-load
- `basePath` is inlined at build time.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath
- `assetPrefix` sets up a CDN and does not cover files in `public/`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix
- `trailingSlash: true` redirects `/about` to `/about/`, except files and `.well-known/`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/trailingSlash
- `reactStrictMode` is on by default with the App Router.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/reactStrictMode
- Dev logging: `logging: { fetches: { fullUrl: true }, incomingRequests: { ignore: [/\/api\/v1\/health/] } }`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/logging

## TypeScript

- Type-check in CI without a build: `next typegen && tsc --noEmit`. `typegen` loads the config in the build phase, so its env vars must exist.
  https://nextjs.org/docs/app/api-reference/cli/next#next-typegen-options
- `next-env.d.ts` is generated: gitignore it, never edit it, put custom declarations in another `.d.ts`.
  https://nextjs.org/docs/app/api-reference/config/typescript#next-envdts
- `typedRoutes: true` (top level) validates literal `href` strings. Non-literal strings need `as Route`.
  https://nextjs.org/docs/app/api-reference/config/typescript#statically-typed-links
- `typescript.ignoreBuildErrors: true` skips the check entirely. Only with `tsc --noEmit` in CI.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/typescript
- By default `next build` runs the project-local `tsc` CLI (opt out with `experimental.useTypeScriptCli: false`), which checks the whole tsconfig project (tests included). `--debug-build-paths` does not narrow it.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/useTypeScriptCli
- In `next.config.ts`, ESM syntax needs Node's native TS resolver (on by default from Node 22.18, opt-in from 22.10). In CommonJS projects prefer `next.config.mts`.
  https://nextjs.org/docs/app/api-reference/config/typescript#using-nodejs-native-typescript-resolver-for-nextconfigts

## ESLint

- Run the ESLint CLI with a flat config:
  ```js
  // eslint.config.mjs
  import { defineConfig, globalIgnores } from 'eslint/config'
  import nextVitals from 'eslint-config-next/core-web-vitals'
  export default defineConfig([...nextVitals, globalIgnores(['.next/**', 'out/**', 'build/**', 'next-env.d.ts'])])
  ```
  https://nextjs.org/docs/app/api-reference/config/eslint#setup-eslint
- Add `eslint-config-next/typescript` for TypeScript and `eslint-config-prettier/flat` with Prettier.
  https://nextjs.org/docs/app/api-reference/config/eslint#with-prettier
- Monorepo: `settings: { next: { rootDir: 'packages/my-app/' } }`.
  https://nextjs.org/docs/app/api-reference/config/eslint#specifying-a-root-directory-within-a-monorepo

## React Compiler

- `reactCompiler: true` is a top-level option, stable since v16.0.0. It needs `babel-plugin-react-compiler` and may slow builds slightly.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler
- Opt-in mode: `reactCompiler: { compilationMode: 'annotation' }` with `'use memo'`. `'use no memo'` opts out.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler#annotations

## Dev speed, memory and Fast Refresh

- Import icons by path, avoid barrel files, and keep Tailwind `content` globs away from `node_modules`.
  https://nextjs.org/docs/app/guides/local-development#2-check-your-imports
- Docker Desktop on Mac or Windows delays file events, so Fast Refresh can take minutes. Run dev on the host.
  https://nextjs.org/docs/app/guides/local-development#2-avoid-common-docker-pitfalls
- On Windows, antivirus software can slow file access: add a folder exclusion for the project or use a Dev Drive.
  https://nextjs.org/docs/app/guides/local-development#1-check-your-computers-antivirus
- Dev may show stale fetch data between HMR refreshes, even for `no-store`. Disable with `experimental.serverComponentsHmrCache: false`.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/serverComponentsHmrCache
- Memory issues in the prerender phase of `next build` (after "Generating static pages"): try `enablePrerenderSourceMaps: false`.
  https://nextjs.org/docs/app/guides/memory-usage
- Fast Refresh falls back to a full reload when an edited file is imported by files outside the React tree (for example a component file that also exports a constant used by a non-React util). Anonymous default exports lose state, so name them. Hooks with dependencies rerun on every refresh, even with `[]`.
  https://nextjs.org/docs/architecture/fast-refresh
- `next dev --inspect` for server debugging. `--inspect-brk` goes through `NODE_OPTIONS`.
  https://nextjs.org/docs/app/guides/debugging

## Testing

- Some tools do not fully support async Server Components (Vitest does not, Cypress component testing does not). Test those end to end.
  https://nextjs.org/docs/app/guides/testing
- Run Playwright E2E tests against production code (`next build` and `next start`) rather than the dev server.
  https://nextjs.org/docs/app/guides/testing/playwright#running-your-playwright-tests
- Jest: wrap the config with `next/jest`, which sets up SWC, mocks CSS, images and `next/font`, and loads `.env*`.
  ```ts
  import nextJest from 'next/jest.js'
  const createJestConfig = nextJest({ dir: './' })
  export default createJestConfig({ testEnvironment: 'jsdom' })
  ```
  https://nextjs.org/docs/app/guides/testing/jest#manual-setup
- Vitest: `@vitejs/plugin-react`, `vite-tsconfig-paths` and `environment: 'jsdom'` in `vitest.config.mts`.
  https://nextjs.org/docs/app/guides/testing/vitest
- Unit-test Proxy matchers with the experimental `unstable_doesProxyMatch` from `next/experimental/testing/server`.
  https://nextjs.org/docs/app/api-reference/file-conventions/proxy#unit-testing-experimental
- `next dev` maintains a block in `AGENTS.md` when it detects a coding agent. `agentRules: false` removes it.
  https://nextjs.org/docs/app/api-reference/config/next-config-js/agentRules
