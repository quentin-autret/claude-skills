# Server Components, 'use client', cache and cacheSignal

## Contents

- Server Components
- 'use client'
- Serializable props
- Third-party libraries
- cache
- cacheSignal
- Taint APIs

## Server Components

- Stable across 19.x minors, but the bundler and framework APIs behind them do not follow semver. Framework authors pin an exact React version or use Canary. https://react.dev/reference/rsc/server-components
- There is no directive for Server Components. `'use server'` marks Server Functions, never components. https://react.dev/reference/rsc/server-components
- They render ahead of time, at build time or per request, and may be `async`. Async components are not supported on the client: use `use` there. https://react.dev/reference/rsc/server-components
- No state and no event handlers. Most Hooks are unavailable. `useId` is not usable in async Server Components. https://react.dev/reference/rsc/use-client and https://react.dev/reference/react/useId
- They cannot create context, but can render a context imported from a `'use client'` module directly: `<UserContext value={user}>`. https://react.dev/reference/rsc/server-components and https://react.dev/blog/2026/09/09/react-19-3
- Await critical data on the server and stream the rest: pass an unawaited Promise to a Client Component under `<Suspense>`. https://react.dev/reference/rsc/server-components

```js
// Server Component
const commentsPromise = db.comments.get(note.id);
<Suspense fallback={<p>Loading Comments...</p>}>
  <Comments commentsPromise={commentsPromise} />
</Suspense>
// Client Component
const comments = use(commentsPromise);
```

https://react.dev/reference/rsc/server-components

## 'use client'

- First line of the file, above imports, comments allowed. Single or double quotes, never backticks. https://react.dev/reference/rsc/use-client
- It marks the module and all its transitive imports as client code. The directive has no effect in a module imported from another client module. https://react.dev/reference/rsc/use-client
- The environment follows the module import tree, not the render tree. A Server Component passed as `children` to a Client Component stays a Server Component. https://react.dev/reference/rsc/use-client
- The same component can be a Server Component in one place and a Client Component in another. https://react.dev/reference/rsc/use-client
- A component without a directive used from a Server Component sends its output, not its code. If the output is much larger than the source (a long SVG path), making it a Client Component can be cheaper. https://react.dev/reference/rsc/use-client
- Server code may import only components or serializable values from a `'use client'` module. Anything else throws. https://react.dev/reference/rsc/use-client

## Serializable props

- Allowed from server to client: string, number, bigint, boolean, `undefined`, `null`, symbols from `Symbol.for`, String, Array, Map, Set, TypedArray, ArrayBuffer, Date, plain objects of these, Server Functions, JSX elements, Promises. https://react.dev/reference/rsc/use-client
- Not allowed: plain functions, classes, class instances, null-prototype objects, `Symbol('x')`. https://react.dev/reference/rsc/use-client
- `Error.cause` and `AggregateError.errors` are transported, and `<Activity>` is supported in Server Components output. https://react.dev/blog/2026/09/09/react-19-3

## Third-party libraries

- A library using `createContext`, Hooks other than `use` and `useId`, `forwardRef`, `memo`, `startTransition` or browser APIs must run on the client. https://react.dev/reference/rsc/use-client
- When it lacks `'use client'`, re-export it from your own `'use client'` file, also needed when it takes event handler props. https://react.dev/reference/rsc/use-client

## cache

- Server Components only. Call `cache(fn)` outside components, in a dedicated module, and import the result everywhere. https://react.dev/reference/react/cache

```js
export default cache(calculateWeekReport);
```

https://react.dev/reference/react/cache

- `cache()` inside a component makes a new cache each render. Each `cache(fn)` call has its own cache, wrapping the same `fn` twice shares nothing. https://react.dev/reference/react/cache
- The cache is reset for every server request. Errors are cached and re-thrown for the same arguments. https://react.dev/reference/react/cache
- Called outside a component (module top level), the function runs without reading or writing the cache. https://react.dev/reference/react/cache
- Arguments compare with `Object.is`: a fresh object always misses. Pass primitives, `calculateNorm(props.x, props.y, props.z)`. https://react.dev/reference/react/cache
- Preload: call the cached async function early without awaiting, then `await` it in the child. https://react.dev/reference/react/cache
- `cache` for shared work and fetches on the server, `useMemo` for computations in Client Components, `memo` to skip re-renders. https://react.dev/reference/react/cache

## cacheSignal

- `cacheSignal()` returns an `AbortSignal` aborted when the render completes, aborts or fails, and `null` outside rendering. Server Components only for now: do not assume it stays `null` on the client. https://react.dev/reference/react/cacheSignal
- `fetch(url, { signal: cacheSignal() })`, ideally with `cache(fetch)`. Called from module top level it returns `null` and never aborts. https://react.dev/reference/react/cacheSignal
- Ignore cancellation errors: `if (!cacheSignal()?.aborted) logError(x)`. https://react.dev/reference/react/cacheSignal

## Taint APIs

- `experimental_taintObjectReference` and `experimental_taintUniqueValue` exist only in `react@experimental`. Do not use them in production. https://react.dev/reference/react/experimental_taintObjectReference
- `experimental_taintObjectReference(message, object)` throws if that exact object reaches a Client Component. Copies and derived values are not tainted. https://react.dev/reference/react/experimental_taintObjectReference
- `experimental_taintUniqueValue(message, lifetime, value)` is for high-entropy strings (tokens, keys). Derived values are not tainted. Never taint low-entropy values such as PINs. https://react.dev/reference/react/experimental_taintUniqueValue
- Pair with `import "server-only"` in files holding secrets. Never rely on tainting alone. https://react.dev/reference/react/experimental_taintUniqueValue
