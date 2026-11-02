# createRoot, hydrateRoot and error reporting

## Contents

- createRoot
- hydrateRoot
- Hydration mismatches
- Error reporting

## createRoot

- `createRoot(domNode, options?)` returns `{ render, unmount }`. Options go to `createRoot`, never to `root.render` ("You passed a second argument to root.render"). https://react.dev/reference/react-dom/client/createRoot
- Not for server-rendered HTML: use `hydrateRoot`. The first `render` clears the container. https://react.dev/reference/react-dom/client/createRoot
- `root.render` is not synchronous: code after it can run before Effects. Wrap in `flushSync` only when initial timing matters. https://react.dev/reference/react-dom/client/createRoot
- After `root.unmount()` the root cannot render again ("Cannot update an unmounted root"). Create a new root. https://react.dev/reference/react-dom/client/createRoot
- Use `createPortal`, not extra roots, for modals. Several roots suit React "sprinkles" in a non-React page. https://react.dev/reference/react-dom/client/createRoot
- "Target container is not a DOM element": `getElementById` returned `null`, or `createRoot(<App />)` was written. "Functions are not valid as a React child": `root.render(App)` instead of `<App />`. https://react.dev/reference/react-dom/client/createRoot
- Render HTML in the browser without `react-dom/server`: `createRoot(div)`, `flushSync(() => root.render(<X />))`, read `div.innerHTML`. https://react.dev/reference/react-dom/server/renderToString

## hydrateRoot

- `hydrateRoot(domNode, reactNode, options?)`. `identifierPrefix` must equal the server one. `formState` must equal the server renderer's. https://react.dev/reference/react-dom/client/hydrateRoot
- Calling `root.render` before hydration ends discards the server HTML and client-renders the whole root. https://react.dev/reference/react-dom/client/hydrateRoot
- Hydrate the whole document with `hydrateRoot(document, <App />)` when App renders `<html>`. https://react.dev/reference/react-dom/client/hydrateRoot
- Unexpected `<head>` and `<body>` tags from extensions or third-party scripts are skipped during hydration. https://react.dev/blog/2024/12/05/react-19

## Hydration mismatches

- Rendered output must equal the server HTML. Treat mismatches as bugs: attribute differences are not guaranteed to be patched, and handlers can attach to the wrong elements. https://react.dev/reference/react-dom/client/hydrateRoot
- Causes: whitespace around the HTML in the root node, `typeof window !== 'undefined'` in render, `window.matchMedia` in render, different data on server and client. https://react.dev/reference/react-dom/client/hydrateRoot
- `Math.random()` and `Date.now()` in render also cause mismatches. https://react.dev/reference/eslint-plugin-react-hooks/lints/purity
- `suppressHydrationWarning={true}` silences one element, one level deep, for unavoidable cases such as timestamps. It does not patch the text. https://react.dev/reference/react-dom/client/hydrateRoot
- Browser-only content: `use(browser())`, with `browser` from `react-dom`, server-renders the nearest `<Suspense>` fallback. It replaces the `isClient` state set in an Effect, which renders twice. https://react.dev/reference/react-dom/client/hydrateRoot and https://react.dev/reference/react-dom/browser

## Error reporting

- Render errors are not re-thrown. Uncaught ones go to `window.reportError`, caught ones (Error Boundary) to `console.error`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Report through root options `onCaughtError`, `onUncaughtError` and `onRecoverableError`, each `(error, errorInfo)` with `errorInfo.componentStack`. A recoverable error may carry `error.cause`. https://react.dev/reference/react-dom/client/createRoot
- React logs every error to the console by default. Production error reporting that relied on errors being re-thrown may need to move to these options. https://react.dev/reference/react-dom/client/createRoot and https://react.dev/blog/2024/04/25/react-19-upgrade-guide

- Error Boundaries do not catch event handlers, server rendering, their own errors, or async code, except errors inside `startTransition` from `useTransition`. https://react.dev/reference/react/Component
- No function-component Error Boundary exists: write a class or use `react-error-boundary`. Place boundaries where a message makes sense (the conversation list, each message), not around each avatar. https://react.dev/reference/react/Component
- `try`/`catch` cannot catch errors of child rendering, only an Error Boundary can. https://react.dev/reference/eslint-plugin-react-hooks/lints/error-boundaries
- In `componentDidCatch`, attach `React.captureOwnerStack()` in development. https://react.dev/reference/react/Component

```js
static getDerivedStateFromError(error) { return { hasError: true }; }
componentDidCatch(error, info) {
  logErrorToMyService(error, info.componentStack, React.captureOwnerStack());
}
```

https://react.dev/reference/react/Component
