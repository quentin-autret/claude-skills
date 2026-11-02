# Transitions, useDeferredValue, use and Suspense

## Contents

- useTransition and startTransition
- useDeferredValue
- use
- lazy
- Suspense

## useTransition and startTransition

- The function passed to `startTransition` is an Action. Name callback props run inside it `action` or `somethingAction`. https://react.dev/reference/react/useTransition
- Async Actions are allowed. `isPending` stays true until every Action completes and the final state shows. https://react.dev/reference/react/useTransition
- State set after an `await` is not a Transition: wrap it in another `startTransition`. https://react.dev/reference/react/useTransition

```js
startTransition(async () => {
  await someAsyncFunction();
  startTransition(() => {
    setPage('/about');
  });
});
```

https://react.dev/reference/react/useTransition

- The callback runs synchronously. A `setTimeout` inside it is not a Transition: put `startTransition` inside the `setTimeout`. https://react.dev/reference/react/useTransition
- A Transition cannot drive a text input. Keep one synchronous state for the input and defer the rest, or use `useDeferredValue`. https://react.dev/reference/react/useTransition
- Transition updates are interrupted by other updates and restarted. With awaits, results can arrive out of order. https://react.dev/reference/react/useTransition
- Do not rely on concurrent Transitions batching together: unrelated Transitions render independently, so a slow one no longer holds up the others. https://react.dev/blog/2026/09/09/react-19-3
- A suspending update inside a Transition keeps the current content instead of showing the fallback. https://react.dev/reference/react/useTransition
- Standalone `startTransition` works outside components but has no `isPending`. Its errors go to `reportError`, not to an Error Boundary. https://react.dev/reference/react/startTransition
- `startTransition` needs the `set` function. For a prop or a custom Hook value, use `useDeferredValue`. https://react.dev/reference/react/startTransition

## useDeferredValue

- `useDeferredValue(value, initialValue?)`: without `initialValue`, the first render is not deferred. https://react.dev/reference/react/useDeferredValue
- Pass primitives or objects created outside render. A new object each render triggers needless background renders. https://react.dev/reference/react/useDeferredValue
- If the background render suspends, the old value stays on screen, not the fallback. https://react.dev/reference/react/useDeferredValue
- It defers display, not fetching: each keystroke still sends a request. Combine with debounce to cut requests. https://react.dev/reference/react/useDeferredValue
- Show staleness with `query !== deferredQuery`. Wrap the slow child in `memo`. https://react.dev/reference/react/useDeferredValue
- Inside a Transition it returns the new value directly. https://react.dev/reference/react/useDeferredValue

## use

- `use` is not a Hook: it may be called in conditions and loops, but only in a component or Hook. https://react.dev/reference/react/use
- Never wrap `use` in `try`/`catch` ("Suspense Exception: This is not a real error!"). A rejected Promise goes to the nearest Error Boundary. https://react.dev/reference/react/use
- The Promise must be the same instance across renders. `use(fetch(...))` or `use(fetchData().then(...))` in render warns "A component was suspended by an uncached promise" and shows the fallback on every render. https://react.dev/reference/react/use
- Create the Promise before rendering (event handler, route loader, Server Component) and pass it down. Without a framework, cache by key in a module-level `Map`. https://react.dev/reference/react/use

```js
const cache = new Map();
export function fetchData(url) {
  if (!cache.has(url)) cache.set(url, getData(url));
  return cache.get(url);
}
```

https://react.dev/reference/react/use

- To refetch, invalidate the entry, start the fetch in `startTransition`, and store the new Promise in state. https://react.dev/reference/react/use
- Server Component to Client Component: pass the Promise as a prop, `await` on the server, `use` on the client. The resolved value must be serializable. https://react.dev/reference/react/use
- `use(context)` is not supported in Server Components. https://react.dev/reference/react/use
- `use(browser(reason?))`, `browser` from `react-dom`, marks a component browser-only. On the server it suspends and the closest `<Suspense>` fallback goes into the HTML. In the browser it returns `undefined`. It needs a Suspense boundary during SSR and a Client Component. https://react.dev/reference/react/use

```js
function BrowserOnly() {
  use(browser('This component requires browser APIs.'));
  return <BrowserContent />;
}
```

https://react.dev/reference/react/use

## lazy

- Declare `lazy` at module level. Inside a component, state resets every render. https://react.dev/reference/react/lazy
- The loader resolves to an object with `.default`: the component must be the default export. The result is cached, a rejection reaches the nearest Error Boundary. https://react.dev/reference/react/lazy

## Suspense

- What suspends: `lazy`, `use(promise)`, `<link rel="stylesheet" precedence>` (up to a timeout), streamed HTML, and fonts and images only when a `<ViewTransition>` animates the reveal (up to 500 ms). Fetching in Effects or handlers does not. https://react.dev/reference/react/Suspense
- `loading="lazy"` or an `onLoad` handler opts an image out of that wait. https://react.dev/reference/react/Suspense
- Reveals are throttled: at most one every 300 ms, boundaries ready in the window reveal together. https://react.dev/reference/react/Suspense
- No state is kept for a tree that suspended before its first mount. https://react.dev/reference/react/Suspense
- Visible content that suspends again shows the fallback, unless the update is in a Transition or uses `useDeferredValue`. Layout Effects of the hidden content are cleaned up and re-run on reveal. https://react.dev/reference/react/Suspense
- A newly mounted boundary still shows its fallback during a Transition. https://react.dev/reference/react/Suspense
- Reset a boundary on navigation with a `key`: `<ProfilePage key={queryParams.id} />`. Without it, old content stays visible during the Transition. https://react.dev/reference/react/Suspense
- Nest boundaries for a reveal sequence. A suspending `fallback` activates the parent boundary. https://react.dev/reference/react/Suspense
- `defer` is experimental: React may show the fallback first even if nothing suspends, for CPU-heavy content. https://react.dev/reference/react/Suspense
- A fallback for a browser-only component: wrap it in `<Suspense>` and call `use(browser())` inside it. https://react.dev/reference/react/Suspense
- A Suspense-enabled router wraps navigations in `startTransition`. https://react.dev/reference/react/Suspense
