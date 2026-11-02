# Effects: when to use them, useEffect, useLayoutEffect, useInsertionEffect

## Contents

- When an Effect is the wrong tool
- useEffect behavior
- Strict Mode and cleanup
- Data fetching
- useLayoutEffect
- useInsertionEffect

## When an Effect is the wrong tool

Use an Effect only for code that runs because the component was displayed and that synchronizes with a system React does not control. https://react.dev/learn/you-might-not-need-an-effect

| Situation | Instead of an Effect |
| --------- | -------------------- |
| Value derived from props or state | Compute during render: `const fullName = first + ' ' + last;` |
| Expensive derived value | `useMemo(() => getFilteredTodos(todos, filter), [todos, filter])` |
| Reset all state when a prop changes | `key={userId}` on the inner component |
| Adjust part of the state | Store an id and derive the item, or the guarded set-during-render pattern |
| Logic shared by two handlers | A function called from both handlers |
| POST on submit, buying a product | Call it in the event handler |
| Chain of Effects setting state | Compute the next state in the handler |
| Notify a parent of a change | Call `onChange` in the same handler as `setIsOn` |
| Pass fetched data to a parent | Fetch in the parent and pass it down |
| Subscribe to an external store | `useSyncExternalStore` |
| App init once per load | A module-level `didInit` flag or `typeof window !== 'undefined'` check |

https://react.dev/learn/you-might-not-need-an-effect

- Chained Effects are fine when each step syncs with the network (dependent dropdowns). https://react.dev/learn/you-might-not-need-an-effect
- Fetching search results from `query` and `page` stays an Effect: the query can come from the URL. An analytics visit on display stays an Effect too. https://react.dev/learn/you-might-not-need-an-effect
- The `set-state-in-effect` lint flags a synchronous `setState` in an Effect: copying props, `setLoading(true)` at the start, transforming data, deriving a selection. Setting state from a measurement in `useLayoutEffect` is valid. https://react.dev/reference/eslint-plugin-react-hooks/lints/set-state-in-effect
- A fade-in is a CSS animation, not an Effect. Heavily coordinated Effects move into a plain JS class that becomes the external system. https://react.dev/learn/reusing-logic-with-custom-hooks

## useEffect behavior

- After each commit with changed dependencies: cleanup with old values, then setup with new values. Final cleanup on unmount. Effects never run during server rendering. https://react.dev/reference/react/useEffect
- No array runs after every commit. `[]` is right when the Effect uses no reactive values. Think start and stop of synchronization, not mount. https://react.dev/learn/lifecycle-of-reactive-effects
- Paint timing: an Effect not caused by an interaction usually runs after paint. An Effect caused by a click may run before paint, so wrap an `alert()` in `setTimeout` to defer it. React may still repaint before processing state set inside the Effect. Use `useLayoutEffect` only to block paint. https://react.dev/reference/react/useEffect
- Think per Effect, not per component lifecycle: an Effect starts synchronizing and later stops, any number of times. https://react.dev/learn/lifecycle-of-reactive-effects
- One Effect per synchronization process. Do not bolt `logVisit(roomId)` into the connection Effect. https://react.dev/learn/lifecycle-of-reactive-effects
- Each render has its own Effect closure over its props and state. https://react.dev/learn/synchronizing-with-effects
- Do not touch a DOM node during render (`ref.current.play()`): call it from an Effect. https://react.dev/learn/synchronizing-with-effects
- No array plus `setCount(count + 1)` in the Effect is an infinite loop. https://react.dev/learn/synchronizing-with-effects
- Different content on server and client: prefer `use(browser())`, with `browser` from `react-dom`, over a `didMount` state set in an Effect, which renders twice. https://react.dev/reference/react-dom/client/hydrateRoot

## Strict Mode and cleanup

- In development React runs one extra setup and cleanup before the first real setup. The user must not see a difference between setup once and setup, cleanup, setup. https://react.dev/reference/react/useEffect
- Ask "how do I fix my Effect so that it works after remounting", never "how to run it once". Do not hide the second run with a ref guard: the connection still leaks on navigation. https://react.dev/learn/synchronizing-with-effects
- Cleanup recipes: nothing for idempotent calls (`map.setZoomLevel`), `dialog.close()` after `showModal()`, remove the same listener, reset an animation, abort or ignore a fetch. Analytics need no fix. https://react.dev/learn/synchronizing-with-effects
- Cleanup without a matching setup (`useEffect(() => () => doSomething(), [])`) is a code smell. https://react.dev/reference/react/useEffect
- Strict Mode below the root skips the extra Effect run on initial mount. Strict Mode also double-invokes Effects during hydration. https://react.dev/reference/react/StrictMode and https://react.dev/blog/2026/09/09/react-19-3
- `<Activity mode="hidden">` destroys Effects and keeps state: Effects must survive hide and show, so keep Strict Mode on. https://react.dev/reference/react/Activity

## Data fetching

- Prefer the framework's data fetching, then a client cache (TanStack Query, useSWR, React Router data APIs). An Effect stays an option when neither fits, with these downsides: no server run, waterfalls, refetch on remount. https://react.dev/reference/react/useEffect

```js
useEffect(() => {
  let ignore = false;
  setBio(null);
  fetchBio(person).then(result => { if (!ignore) setBio(result); });
  return () => { ignore = true; };
}, [person]);
```

https://react.dev/reference/react/useEffect

- Wrap fetching in a custom Hook (`useData(url)`) so the strategy can change later. https://react.dev/learn/you-might-not-need-an-effect
- Suspense does not detect fetching inside Effects or event handlers. https://react.dev/reference/react/Suspense

## useLayoutEffect

- Runs after the DOM commit and before paint. Its code and every state update inside it block paint. "Prefer useEffect when possible." https://react.dev/reference/react/useLayoutEffect
- A state update inside `useLayoutEffect` runs all remaining Effects, `useEffect` included, immediately. https://react.dev/reference/react/useLayoutEffect
- Use it to measure then re-render before paint (tooltip placement). https://react.dev/reference/react/useLayoutEffect

```js
useLayoutEffect(() => {
  const { height } = ref.current.getBoundingClientRect();
  setTooltipHeight(height);
}, []);
```

https://react.dev/reference/react/useLayoutEffect

- It does nothing on the server. For a layout-dependent component, call `use(browser())` so the nearest `<Suspense>` fallback is server-rendered. https://react.dev/reference/react/useLayoutEffect

## useInsertionEffect

- Only for CSS-in-JS library authors. It runs before layout Effects to inject `<style>`. https://react.dev/reference/react/useInsertionEffect
- Client only, cannot update state, refs not attached yet, may run before or after the DOM update. Cleanup and setup interleave one component at a time. https://react.dev/reference/react/useInsertionEffect
- React recommends static CSS files plus inline styles for dynamic values over runtime `<style>` injection. https://react.dev/reference/react/useInsertionEffect
