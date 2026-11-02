# Effect dependencies, useEffectEvent and custom Hooks

## Contents

- Dependencies are not a choice
- Removing a dependency
- useEffectEvent
- Custom Hooks
- exhaustive-deps configuration

## Dependencies are not a choice

- Reactive values are props, state, context values and everything declared in the component body, including values computed from them. Module-level constants and values declared inside the Effect are not. https://react.dev/learn/lifecycle-of-reactive-effects
- To drop a dependency, change the code so the value is no longer reactive, then follow the linter. "If you want to change the dependencies, change the surrounding code first." https://react.dev/learn/removing-effect-dependencies
- Never suppress `react-hooks/exhaustive-deps`: "All errors flagged by the linter are legitimate." Suppressing it can leave stale values. https://react.dev/learn/lifecycle-of-reactive-effects and https://react.dev/learn/removing-effect-dependencies
- `set` functions and `useRef` objects are stable and may be omitted, only when the linter can see it. A ref received from a parent must be listed. `ref.current` and `location.pathname` cannot be dependencies (use `useSyncExternalStore` for globals). https://react.dev/learn/synchronizing-with-effects and https://react.dev/learn/lifecycle-of-reactive-effects
- `startTransition` from `useTransition` and `dispatchAction` from `useActionState` are stable too. https://react.dev/reference/react/useTransition and https://react.dev/reference/react/useActionState
- Debug a dependency that changes every render: `console.log([serverUrl, roomId])`, store both arrays as globals, compare each index with `Object.is`. https://react.dev/reference/react/useEffect

## Removing a dependency

Ask in this order. https://react.dev/learn/removing-effect-dependencies

1. Should the code run in an event handler instead? Move it there.
2. Does the Effect do unrelated things? Split it, one Effect per synchronization process.
3. Does it read state to compute the next state? Use an updater: `setMessages(msgs => [...msgs, msg])`.
4. Does it read a value without reacting to it? Use an Effect Event.
5. Does an object or function change identity every render? See below.

- Static object or function: move it outside the component. Dynamic: create it inside the Effect and depend on primitives (`roomId`). https://react.dev/learn/removing-effect-dependencies
- Object prop: destructure primitives before the Effect, `const { roomId, serverUrl } = options;`. Function prop: call it before the Effect if pure, wrap it in an Effect Event if it is a handler. https://react.dev/learn/removing-effect-dependencies
- `useMemo` or `useCallback` on a dependency is the last resort, after updaters, moving the value inside and Effect Events. https://react.dev/reference/react/useEffect
- A truly one-time side effect keeps its dependency and guards with a ref. Use this only when the side effect must not repeat, never to hide a missing cleanup. https://react.dev/reference/eslint-plugin-react-hooks/lints/exhaustive-deps and https://react.dev/learn/synchronizing-with-effects

```js
const sent = useRef(false);
useEffect(() => {
  if (sent.current) return;
  sent.current = true;
  sendAnalytics(userId);
}, [userId]);
```

https://react.dev/reference/eslint-plugin-react-hooks/lints/exhaustive-deps

## useEffectEvent

- `useEffectEvent(callback)` returns a function that always sees the latest committed props and state. It is never a dependency. https://react.dev/reference/react/useEffectEvent

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

- Call it only from `useEffect`, `useLayoutEffect`, `useInsertionEffect` or another Effect Event of the same component. Never during render, never passed to a child or another Hook. For handlers and children, use a regular function or `useCallback`. https://react.dev/reference/react/useEffectEvent
- Its identity changes every render on purpose: listing it as a dependency re-runs the Effect every render. https://react.dev/reference/react/useEffectEvent
- Never use it to hide a dependency. `useEffectEvent(() => log(pageUrl))` called from `useEffect(..., [])` misses page changes. https://react.dev/reference/react/useEffectEvent
- Pass the reactive part as an argument: `onVisit(url)` with `useEffectEvent(visitedUrl => logVisit(visitedUrl, numberOfItems))`. `url` stays a dependency, and in async code the argument is the value that triggered the Effect. https://react.dev/learn/separating-events-from-effects
- A handler received by a custom Hook is wrapped inside the Hook: `const onMessage = useEffectEvent(onReceiveMessage);`, deps `[roomId, serverUrl]`. Declare Effect Events next to the Effect that uses them, never pass one into a Hook such as `useTimer(onTick, 1000)`. https://react.dev/learn/reusing-logic-with-custom-hooks and https://react.dev/learn/separating-events-from-effects
- Lint messages: "A function wrapped in useEffectEvent can't be called during rendering", "Functions returned from useEffectEvent must not be included in the dependency array". https://react.dev/reference/react/useEffectEvent
- It needs a current `eslint-plugin-react-hooks`. https://react.dev/blog/2025/10/01/react-19-2

## Custom Hooks

- A Hook name is `use` plus a capital letter. A function that calls no Hook is not named `use...` (`useSorted` becomes `getSorted`). https://react.dev/learn/reusing-logic-with-custom-hooks
- Custom Hooks share logic, not state: each call has its own state. https://react.dev/learn/reusing-logic-with-custom-hooks
- Name Hooks after concrete use cases: `useData(url)`, `useChatRoom(options)`, `useImpressionLog(eventName, extraData)`. Avoid `useMount`, `useEffectOnce`, `useUpdateEffect`: the linter only checks direct `useEffect` calls, so they hide missing dependencies. https://react.dev/learn/reusing-logic-with-custom-hooks
- It is recommended to wrap the functions a custom Hook returns in `useCallback`. https://react.dev/reference/react/useCallback
- Wrapping a single `useState` in a Hook is probably unnecessary. Wrapping Effects pays off: "The fewer raw useEffect calls you have in your components, the easier you will find to maintain your application." https://react.dev/learn/reusing-logic-with-custom-hooks and https://react.dev/learn/you-might-not-need-an-effect
- Do not write higher-order Hooks (`withLogging(useData)`) or pass a Hook as a prop (`<Button useData={useDataWithLogging} />`). https://react.dev/reference/rules/react-calls-components-and-hooks

## exhaustive-deps configuration

- The shared setting `settings["react-hooks"].additionalEffectHooks` (regex) makes custom Hooks count as Effects for `exhaustive-deps` and `rules-of-hooks`, so `useEffectEvent` may be called from them. https://react.dev/reference/eslint-plugin-react-hooks/lints/rules-of-hooks

```json
{ "settings": { "react-hooks": { "additionalEffectHooks": "(useMyEffect|useCustomEffect)" } } }
```

https://react.dev/reference/eslint-plugin-react-hooks/lints/rules-of-hooks

- The older rule option `additionalHooks` still works and takes precedence over the shared setting. https://react.dev/reference/eslint-plugin-react-hooks/lints/exhaustive-deps
