---
name: react-docs
description: Writes, reviews and upgrades React 19.3 code from the official react.dev documentation. Use when writing components and Hooks, removing unnecessary useEffect calls, fixing Effect dependencies with useEffectEvent, building forms with Actions (useActionState, useOptimistic, useFormStatus, form action), using Server Components, 'use client', 'use server' and Server Functions, Suspense, use(), useTransition, useDeferredValue, ViewTransition, Activity, adopting React Compiler and eslint-plugin-react-hooks, deciding on useMemo, useCallback and memo, server rendering and hydration (createRoot, hydrateRoot, renderToPipeableStream, prerender), or migrating from React 18 (forwardRef, Context.Provider, ReactDOM.render, propTypes, defaultProps). Triggers on React, JSX, hooks, useState, useEffect, useRef, useContext, Server Components, RSC, Actions, React Compiler, Strict Mode, hydration mismatch, React 19 upgrade.
metadata:
  version: "19.3"
  source: https://react.dev/reference/react
  read: "2026-11-02"
---

# React 19.3

## Overview

This skill holds the React 19.3 rules that are easy to get wrong: Effects that should not exist, dependency traps, Actions and their Transitions, Server Components boundaries, React Compiler adoption, and the APIs removed or deprecated in 19. Every claim carries its page URL.

## Version covered

React 19.3, the release react.dev documents. Previous majors live on 18.react.dev and older sites. https://react.dev/versions

Read the quick start, then open the one reference file that matches the task.

## Quick start

- Use an Effect only to synchronize with a system React does not control. Derive values during render, reset state with a `key`, and run user-caused logic in event handlers. https://react.dev/learn/you-might-not-need-an-effect
- Dependencies follow the code: never suppress `react-hooks/exhaustive-deps`. To drop one, move objects and functions inside the Effect, use an updater, or read the value from an Effect Event. https://react.dev/learn/removing-effect-dependencies

```js
const onConnected = useEffectEvent(() => {
  showNotification('Connected!', theme);
});
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.on('connected', () => { onConnected(); });
  connection.connect();
  return () => connection.disconnect();
}, [roomId]);
```

https://react.dev/learn/separating-events-from-effects

- An Effect must survive setup, cleanup, setup: Strict Mode runs that in development. Fix the cleanup, never hide the second run with a ref guard. https://react.dev/learn/synchronizing-with-effects
- `ref` is a prop of function components. Render `<Context value={x}>` as the provider. A ref callback may return its cleanup. https://react.dev/reference/react/useRef and https://react.dev/reference/react/createContext and https://react.dev/reference/react-dom/components/common
- State set after an `await` inside `startTransition` is not a Transition: wrap it in another `startTransition`. https://react.dev/reference/react/useTransition
- With `<form action>`, the `useActionState` reducer receives `(prevState, formData)`. Call `dispatchAction` from an Action: a `<form action>` or `startTransition`. https://react.dev/reference/react/useActionState
- `useFormStatus` reads the parent `<form>` only: call it in a child component. https://react.dev/reference/react-dom/hooks/useFormStatus
- `use(promise)` needs a cached Promise: one created during render is recreated on every render. Never wrap `use` in `try`/`catch`: use an Error Boundary. https://react.dev/reference/react/use
- `'use server'` marks Server Functions, never Server Components, which have no directive. Validate every Server Function argument and check authorization. https://react.dev/reference/rsc/server-components and https://react.dev/reference/rsc/use-server
- Browser-only component: `use(browser())`, `browser` from `react-dom`, under a `<Suspense>` boundary, instead of `typeof window` or a mounted flag. https://react.dev/reference/react-dom/browser
- With React Compiler, write plain code and keep `useMemo` or `useCallback` only for precise control, such as an Effect dependency. Leave existing memoization in place. https://react.dev/learn/react-compiler/introduction
- Lint with `eslint-plugin-react-hooks` `recommended`. It carries the compiler rules, so remove `eslint-plugin-react-compiler`. https://react.dev/blog/2025/10/07/react-compiler-1
- Define components at module level, never inside another component. https://react.dev/reference/eslint-plugin-react-hooks/lints/static-components
- Never put a number left of `&&`: `count && <p/>` renders `0`. https://react.dev/learn/conditional-rendering

## Reference files

| File | Open it for |
| ---- | ----------- |
| [references/state.md](references/state.md) | `useState`, updaters, set during render, `useReducer`, immutable updates, state structure, `key` resets |
| [references/refs-and-context.md](references/refs-and-context.md) | `useRef`, ref callbacks, `ref` as a prop, `useImperativeHandle`, context, `useId`, `useSyncExternalStore` |
| [references/effects.md](references/effects.md) | When not to use an Effect, `useEffect` timing, Strict Mode cleanup, fetching, `useLayoutEffect`, `useInsertionEffect` |
| [references/effect-dependencies.md](references/effect-dependencies.md) | Removing dependencies, `useEffectEvent`, custom Hooks, `additionalEffectHooks` |
| [references/memoization.md](references/memoization.md) | `useMemo`, `useCallback`, `memo`, with and without the compiler |
| [references/react-compiler.md](references/react-compiler.md) | Installing, options, directives, incremental adoption, debugging the compiler |
| [references/rules-and-lints.md](references/rules-and-lints.md) | Rules of React, Rules of Hooks, `eslint-plugin-react-hooks` rules, Strict Mode |
| [references/components-and-tooling.md](references/components-and-tooling.md) | Keys, conditional rendering, Fragment refs, Profiler, `act`, TypeScript, DevTools |
| [references/transitions-and-suspense.md](references/transitions-and-suspense.md) | `useTransition`, `startTransition`, `useDeferredValue`, `use`, `lazy`, `<Suspense>` |
| [references/view-transitions-and-activity.md](references/view-transitions-and-activity.md) | `<ViewTransition>`, `addTransitionType`, `<Activity>` |
| [references/actions-and-optimistic.md](references/actions-and-optimistic.md) | `useActionState`, `useOptimistic`, `useFormStatus`, `<form action>` |
| [references/form-elements-and-events.md](references/form-elements-and-events.md) | Controlled inputs, `<select>`, `<textarea>`, common props, events, custom elements |
| [references/server-components.md](references/server-components.md) | Server Components, `'use client'`, serializable props, `cache`, `cacheSignal`, taint |
| [references/server-functions.md](references/server-functions.md) | `'use server'`, calling Server Functions, permalink, security |
| [references/client-roots-and-hydration.md](references/client-roots-and-hydration.md) | `createRoot`, `hydrateRoot`, mismatches, error reporting, Error Boundaries |
| [references/server-rendering.md](references/server-rendering.md) | `renderToPipeableStream`, `renderToReadableStream`, status codes, `prerender`, `resume` |
| [references/dom-apis-and-resources.md](references/dom-apis-and-resources.md) | `browser`, `createPortal`, `flushSync`, `preload`, `preinit`, `<img>`, `<link>`, `<script>`, `<title>` |
| [references/upgrading-to-19.md](references/upgrading-to-19.md) | Upgrade steps, codemods, removed and deprecated APIs, TypeScript changes, 19.2 and 19.3 |
| [references/legacy-apis.md](references/legacy-apis.md) | `Children`, `cloneElement`, class components, `forwardRef`, `PureComponent` |

## Old patterns

| Old usage | Replacement | Source |
| --------- | ----------- | ------ |
| `forwardRef((props, ref) => ...)` | `function MyInput({ ref, ...props })` | https://react.dev/reference/react/forwardRef |
| `<Context.Provider value={x}>` | `<Context value={x}>` | https://react.dev/reference/react/createContext |
| `<Context.Consumer>` render prop | `useContext(Context)` or `use(Context)` | https://react.dev/reference/react/createContext |
| `ReactDOM.useFormState` | `useActionState` from `react` | https://react.dev/blog/2024/12/05/react-19 |
| `ReactDOM.render`, `ReactDOM.hydrate` | `createRoot(el).render()`, `hydrateRoot` | https://react.dev/reference/react-dom |
| `unmountComponentAtNode` | `root.unmount()` | https://react.dev/reference/react-dom |
| `findDOMNode` | DOM refs | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| String refs, `this.refs` | Ref callbacks or `createRef` | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| `contextTypes`, `getChildContext` | `createContext`, `static contextType` | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| `element.ref` | `element.props.ref` | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| `react-test-renderer`, shallow rendering | Testing Library | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| Global `JSX` namespace in types | `declare module "react" { namespace JSX {} }` | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| UMD build script tag | ESM CDN such as esm.sh | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| `propTypes` | TypeScript | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| `defaultProps` on a function component | ES default parameters | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| `import { act } from 'react-dom/test-utils'` | `import { act } from 'react'` | https://react.dev/blog/2024/04/25/react-19-upgrade-guide |
| Synchronous `act(() => ...)` | `await act(async () => ...)` | https://react.dev/reference/react/act |
| Ref callback relying on a `null` call | Return a cleanup function | https://react.dev/reference/react-dom/components/common |
| `renderToString` for SSR | `renderToPipeableStream` or `renderToReadableStream` | https://react.dev/reference/react-dom/server/renderToString |
| `renderToNodeStream`, `renderToStaticNodeStream` | `react-dom/server` streaming APIs | https://react.dev/reference/react-dom |
| Throwing on the server to skip SSR | `use(browser())` | https://react.dev/reference/react/Suspense |
| `isClient` state set in an Effect, `typeof window` in render | `use(browser())` under `<Suspense>` | https://react.dev/reference/react-dom/browser |
| `eslint-plugin-react-compiler` | `eslint-plugin-react-hooks` | https://react.dev/blog/2025/10/07/react-compiler-1 |
| `"use no forget"` | `"use no memo"` | https://react.dev/reference/react-compiler/directives/use-no-memo |
| `react({ babel: { plugins: [...] } })` on `@vitejs/plugin-react` 6 | `reactCompilerPreset` with `@rolldown/plugin-babel` | https://react.dev/learn/react-compiler/installation |
| `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` | `UNSAFE_` names, or the page's alternatives | https://react.dev/reference/react/Component |
