# Server and Client Components

The `"use client"` boundary, serialization, interleaving, environment poisoning, error handling and optimistic UI in the Next.js 16.4.4 App Router.

## Contents

- Defaults and project setup
- Placing the `"use client"` boundary
- Crossing the boundary
- Rendering environments
- Keeping server code on the server
- Error handling
- Transitions and optimistic UI

## Defaults and project setup

- Layouts and pages are Server Components by default.
  https://nextjs.org/docs/app/getting-started/server-and-client-components
- Minimums: Node.js 20.9. Browsers: Chrome, Edge and Firefox 111+, Safari 16.4+.
  https://nextjs.org/docs/app/getting-started/installation#system-requirements
- Minimum TypeScript version: 5.1.0.
  https://nextjs.org/docs/app/getting-started/installation#set-up-typescript
- Still declare `react` and `react-dom` in `package.json`, even though the App Router ships React canary internally.
  https://nextjs.org/docs/app/getting-started/installation#manual-installation
- A folder is a public route only when it holds a `page` or `route` file. `_folder` opts out of routing, `%5F` makes a URL segment that starts with an underscore.
  https://nextjs.org/docs/app/getting-started/project-structure#private-folders
  https://nextjs.org/docs/app/getting-started/project-structure#nested-routes
- `proxy.ts` and `instrumentation.ts` are top-level files. With a `src` folder, `proxy.ts` goes inside `src`.
  https://nextjs.org/docs/app/getting-started/project-structure#top-level-files
  https://nextjs.org/docs/app/api-reference/file-conventions/src-folder

## Placing the `"use client"` boundary

- Push `'use client'` down to the interactive leaf (a `<Search />` inside an otherwise server layout) to cut bundle size.
  https://nextjs.org/docs/app/getting-started/server-and-client-components#reducing-js-bundle-size
- `'use client'` goes at the top of the file, before imports, and covers the whole module. It cannot sit inside a function.
  https://nextjs.org/docs/app/api-reference/directives/use-client#usage
- Once a file has the directive, its imports and the components it directly renders join the client bundle, so not every client file needs it. This does not apply to Server Components passed as children or other props.
  https://nextjs.org/docs/app/getting-started/server-and-client-components#using-client-components
- A third-party client-only component without the directive: re-export it from your own `'use client'` file.
  https://nextjs.org/docs/app/getting-started/server-and-client-components#third-party-components
- Context does not work in Server Components. Write a `'use client'` provider taking `children` and render it as deep as possible (around `{children}`, not `<html>`).
  https://nextjs.org/docs/app/getting-started/server-and-client-components#context-providers
- Native HTML needs no Client Component: `<details>`, `<form action={serverAction}>`, `<video controls>`.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#state-and-interactivity

## Crossing the boundary

- Props to Client Components must be serializable. A function prop such as `onClick` throws. Server Actions cross as references.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#crossing-the-boundary
- The TS plugin accepts a function prop only if it is named `action` or ends in `Action`.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#crossing-the-boundary
- Interleaving: pass a Server Component as `children` (or any prop) to a Client Component. The owner who writes the JSX decides where it renders, not the parent.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#crossing-the-boundary
- Compound components with static members (`Menu.Item`) break when imported into a Server Component: the value is a client reference and `Menu.Item` is `undefined` ("Element type is invalid"). Use named exports.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#crossing-the-boundary
- Stream server data into a Client Component: start the request in the Server Component, pass the pending promise as a prop, read it with `use()` under `<Suspense>`.
  ```tsx
  const posts = getPosts() // no await
  <Suspense fallback={<p>Loading...</p>}><Posts posts={posts} /></Suspense>
  // client: const allPosts = use(posts)
  ```
  https://nextjs.org/docs/app/getting-started/fetching-data#streaming-data-with-the-use-api
- Every prop passed to a Client Component is serialized into the page. Be deliberate about what you pass.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#how-data-enters-the-tree

## Rendering environments

- Client Components render on the server for the first HTML and again in the browser for hydration, so a `console.log` shows in both. On client navigation they render in the browser only.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#rendering-environments
- "Server-rendered" says how the HTML was produced. "Server Component" says where the code runs and that it never ships.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#rendering-environments
- Content gated behind interaction is absent from the first HTML that crawlers see.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#rendering-environments
- Static versus dynamic is decided per component, not per route: a static shell, cached functions and streamed dynamic parts coexist.
  https://nextjs.org/docs/app/guides/rendering-philosophy#static-and-dynamic-as-a-spectrum
- Do not mutate Server Component DOM directly. Re-render on the server instead.
  https://nextjs.org/docs/app/guides/server-and-client-boundary#state-and-interactivity

## Keeping server code on the server

- Only `NEXT_PUBLIC_` variables reach the client. Others become an empty string in client code.
  https://nextjs.org/docs/app/getting-started/server-and-client-components#preventing-environment-poisoning
- `import 'server-only'` turns an accidental client import into a build error. `client-only` guards `window`-dependent code. Installing the packages is optional: Next.js resolves the imports itself.
  ```ts
  import 'server-only'
  export async function getData() { /* uses process.env.API_KEY */ }
  ```
  https://nextjs.org/docs/app/getting-started/server-and-client-components#preventing-environment-poisoning

## Error handling

- Model expected errors (validation, failed requests) as return values, not throws. Show them with `useActionState` in an `aria-live="polite"` element.
  https://nextjs.org/docs/app/getting-started/error-handling#handling-expected-errors
- Uncaught exceptions reach the nearest `error.tsx`, which must be `'use client'`. It receives `{ error, retry }`. `retry()` re-fetches and re-renders the segment, `reset()` clears the error state and re-renders the children without re-fetching (use `retry()` in most cases).
  https://nextjs.org/docs/app/getting-started/error-handling#nested-error-boundaries
  https://nextjs.org/docs/app/api-reference/file-conventions/error#reset
- `error` is typed `unknown`: a value thrown in a Client Component is passed as-is and may not be an `Error`. Narrow with `instanceof Error` before reading `message` or `digest`.
  https://nextjs.org/docs/app/api-reference/file-conventions/error#error
- `catchError` from `next/error` builds component-level boundaries: `export default catchError(Fallback)` with `Fallback(props, { error, retry })`.
  https://nextjs.org/docs/app/getting-started/error-handling#nested-error-boundaries
- Boundaries miss errors in event handlers and in async code after render. Catch those and keep them in state. Errors thrown inside `startTransition` do bubble.
  https://nextjs.org/docs/app/getting-started/error-handling#nested-error-boundaries
- `global-error.tsx` replaces the root layout when active, so it renders its own `<html>` and `<body>`.
  https://nextjs.org/docs/app/getting-started/error-handling#global-errors

## Transitions and optimistic UI

- Stream slow reads: keep the page synchronous and wrap each async section in its own `<Suspense>`. Use `params.then(...)` inline to avoid blocking on params.
  https://nextjs.org/docs/app/guides/interactive-apps#step-1-stream-slow-data-with-suspense
- Optimistic toggle: `useOptimistic` plus `startTransition` calling the Server Action. Read the optimistic value, not the prop, so rapid clicks stay correct.
  https://nextjs.org/docs/app/guides/interactive-apps#step-2-respond-instantly-to-a-toggle
- Optimistic lists: keep only pending items in `useOptimistic([])`. The Server Component renders the persisted list.
  https://nextjs.org/docs/app/guides/interactive-apps#step-4-add-comments-with-instant-feedback
- State updates after an `await` inside a transition are not part of it. Wrap them: `startTransition(() => setIsOpen(false))`.
  https://nextjs.org/docs/app/guides/interactive-apps#step-6-manage-form-lifecycle
- Expose pending state to ancestors with a `data-pending` attribute and CSS `:has()`. Prefer client state when a broad `:has()` anchor toggles on high-frequency interactions like dragging or scrolling.
  https://nextjs.org/docs/app/guides/interactive-apps#step-3-filter-with-pending-feedback
- Use a standalone `startTransition` instead of `useTransition` when `isPending` would trigger unwanted UI.
  https://nextjs.org/docs/app/guides/interactive-apps#step-5-move-cards-between-columns
