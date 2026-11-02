# Lists, conditions, Fragment, testing, TypeScript and DevTools

## Contents

- Lists and keys
- Conditional rendering
- Fragment and Fragment refs
- Profiler and Performance tracks
- act and captureOwnerStack
- TypeScript
- react-devtools-cdt-mcp

## Lists and keys

- Do not generate keys on the fly. `key={Math.random()}` recreates every item and loses input. https://react.dev/learn/rendering-lists
- Index keys break on insert, delete and reorder. Use them only for lists that never reorder. Store keys in the data: database ids, a counter, or `crypto.randomUUID()` at creation. https://react.dev/learn/rendering-lists
- `key` is not a prop: `<Profile key={id} userId={id} />`. https://react.dev/learn/rendering-lists
- Keep the `key` on the element in the array (`<Recipe key={recipe.id} />`), not on the root inside the component. https://react.dev/learn/rendering-lists
- `<>` cannot take a key: use `<Fragment key={person.id}>`. https://react.dev/learn/rendering-lists
- A block-bodied arrow `=> {` in `map` needs an explicit `return`. https://react.dev/learn/rendering-lists

## Conditional rendering

- Never put a number left of `&&`: `count && <p/>` renders `0`. Write `count > 0 && ...`. https://react.dev/learn/conditional-rendering
- Prefer including the component conditionally in the parent over returning `null` from it. https://react.dev/learn/conditional-rendering

## Fragment and Fragment refs

- `<Fragment ref={fragmentRef}>` gives a `FragmentInstance` without a wrapper `div`. https://react.dev/blog/2026/09/09/react-19-3
- Methods: `addEventListener`, `removeEventListener`, `dispatchEvent`, `focus`, `focusLast`, `blur`, `observeUsing`, `unobserveUsing`, `getClientRects`, `getRootNode`, `compareDocumentPosition`, `scrollIntoView`. https://react.dev/reference/react/Fragment
- Event, observer and rect methods target first-level DOM children only. Components are looked through, nested DOM children are not. `focus` and `focusLast` search depth-first. https://react.dev/reference/react/Fragment
- `scrollIntoView(alignToTop?)` takes a boolean only: an options object throws. https://react.dev/reference/react/Fragment
- `observeUsing` ignores text nodes. Listeners do not apply inside a hidden `<Activity>` until it shows. https://react.dev/reference/react/Fragment
- React adds and removes listeners on children added or removed later. https://react.dev/reference/react/Fragment

```js
useEffect(() => {
  const fragmentInstance = fragmentRef.current;
  if (fragmentInstance === null) return;
  fragmentInstance.addEventListener('click', onClick);
  return () => fragmentInstance.removeEventListener('click', onClick);
}, [onClick]);
```

https://react.dev/reference/react/Fragment

## Profiler and Performance tracks

- `<Profiler id onRender>` is off in production builds: use a profiling build (`react-dom/profiling`, best through a bundler alias). Profiling adds some overhead. https://react.dev/reference/react/Profiler and https://react.dev/reference/dev-tools/react-performance-tracks
- Compare `actualDuration`, which should drop after mount when memoization works, with `baseDuration`. https://react.dev/reference/react/Profiler
- React Performance tracks appear in the browser Performance panel in development and profiling builds. The Scheduler track splits into Blocking, Transition, Suspense and Idle. https://react.dev/reference/dev-tools/react-performance-tracks
- "Cascading update" entries (an update scheduled during a render pass, which can make React discard completed work) are a pattern for performance regressions. In development, clicking one shows the component that scheduled it, and clicking a render entry shows the changed props. https://react.dev/reference/dev-tools/react-performance-tracks
- In profiling builds the Components track lists only `<Profiler>` subtrees, unless the React DevTools extension is enabled. https://react.dev/reference/dev-tools/react-performance-tracks

## act and captureOwnerStack

- Use `await act(async () => { ... })`. The sync form will be deprecated and removed. Import it from `react`. https://react.dev/reference/react/act
- Set `global.IS_REACT_ACT_ENVIRONMENT = true`, or React warns "The current testing environment is not configured to support act(...)". React Testing Library sets it. https://react.dev/reference/react/act
- Dispatched DOM events only work on a container attached to `document`. https://react.dev/reference/react/act
- `captureOwnerStack()` works in development only and is not exported in production. In shared files use `import * as React from 'react'` and guard with `process.env.NODE_ENV !== 'production'`. https://react.dev/reference/react/captureOwnerStack
- It returns `null` in `setTimeout`, after `fetch` and in custom DOM listeners: capture it earlier. https://react.dev/reference/react/captureOwnerStack

## TypeScript

- Install `@types/react` and `@types/react-dom`. Use `.tsx` and a valid `jsx` option (`preserve` for most apps). https://react.dev/learn/typescript
- `useState<Status>("idle")` for unions. Children: `React.ReactNode` (broad) or `React.ReactElement` (elements only). TypeScript cannot restrict children to `<li>` elements. https://react.dev/learn/typescript
- Type extracted handlers: `React.ChangeEvent<HTMLInputElement>`, read `event.currentTarget.value`. Styles: `React.CSSProperties`. https://react.dev/learn/typescript
- `ReactElement["props"]` defaults to `unknown`. The global `JSX` namespace is gone: augment `declare module "react" { namespace JSX { ... } }`. The module specifier follows the `jsx` option: `react/jsx-runtime` for `react-jsx`, `react/jsx-dev-runtime` for `react-jsxdev`, `react` for `react` and `preserve`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- `useReducer`: no type arguments, or `useReducer<State, [Action]>(reducer)`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide

## react-devtools-cdt-mcp

- An experimental browser library, not an MCP server. It registers React tools that Chrome DevTools MCP discovers. https://react.dev/reference/dev-tools/react-devtools-cdt-mcp
- `npm install --save-dev react-devtools-cdt-mcp`, then `import 'react-devtools-cdt-mcp/register';` as the very first import, before anything loads `react-dom`. Imported later, the component tree is empty. https://react.dev/reference/dev-tools/react-devtools-cdt-mcp
- Development builds only: it reports every commit and slows the app. It throws outside a browser (SSR, tests). https://react.dev/reference/dev-tools/react-devtools-cdt-mcp
- Tools include `react_get_component_tree`, `react_find_components`, `react_get_owner_stack`, `react_get_component_source`, `react_start_profiling`, `react_stop_profiling`, `react_get_commit_report`. Uids such as `"r5"` are stable until reload. https://react.dev/reference/dev-tools/react-devtools-cdt-mcp
