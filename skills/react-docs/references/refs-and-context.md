# Refs, context, useId and external stores

## Contents

- useRef
- DOM refs and ref callbacks
- ref as a prop and useImperativeHandle
- Context
- useId
- useSyncExternalStore
- useDebugValue

## useRef

- Do not read or write `ref.current` during render, except for lazy initialization. Read and write it in event handlers and Effects. Anything displayed goes in state: `{ref.current}` in JSX never updates. https://react.dev/reference/react/useRef

```js
const playerRef = useRef(null);
if (playerRef.current === null) {
  playerRef.current = new VideoPlayer();
}
```

https://react.dev/reference/react/useRef

- `useRef(new VideoPlayer())` builds the object on every render, only the first is kept. https://react.dev/reference/react/useRef
- The `refs` lint treats as a ref any value from `useRef()` or `createRef()`, any identifier named `ref` or ending in `Ref` whose `.current` is used, and anything passed to a JSX `ref`. A plain object named `fooRef` with `.current` is flagged: rename it. https://react.dev/reference/eslint-plugin-react-hooks/lints/refs
- Strict Mode creates each ref object twice in development and discards one. https://react.dev/reference/react/useRef
- TypeScript: `useRef` requires an argument. `useRef<T>(null)` returns `RefObject<T | null>`, all refs are mutable, `MutableRef` is deprecated. https://react.dev/blog/2024/04/25/react-19-upgrade-guide

## DOM refs and ref callbacks

- React sets `current` at commit and back to `null` on removal. Refs are `null` during the first render, so use them from handlers and Effects. https://react.dev/learn/manipulating-the-dom-with-refs
- A ref callback returns a cleanup function, called when the ref is detached. Without a cleanup, React still calls the callback with `null`, a behavior to be removed. https://react.dev/reference/react-dom/components/common
- Refs for a list: one ref callback writing into a `Map`, since `useRef` cannot be called in `map`. https://react.dev/learn/manipulating-the-dom-with-refs

```js
ref={node => {
  map.set(cat, node);
  return () => {
    map.delete(cat);
  };
}}
```

https://react.dev/learn/manipulating-the-dom-with-refs

- An inline arrow ref is a new function each render: React cleans up the old one and calls the new one each render. Strict Mode runs one extra setup and cleanup of ref callbacks in development. https://react.dev/reference/react-dom/components/common
- TypeScript: a ref callback must not implicitly return. `ref={current => {instance = current}}` with braces, not `ref={current => (instance = current)}`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Scroll to a new item: `flushSync(() => setTodos([...]))`, then `listRef.current.lastChild.scrollIntoView()`. Without `flushSync` the DOM is not updated yet. https://react.dev/learn/manipulating-the-dom-with-refs
- Avoid modifying, adding children to or removing children from DOM nodes React manages (`element.remove()` then a state change crashes). Non-destructive actions like focus and scroll are safe. https://react.dev/learn/manipulating-the-dom-with-refs

## ref as a prop and useImperativeHandle

- Function components receive `ref` as a regular prop. "Cannot read properties of null" on a custom component means it does not forward it. https://react.dev/reference/react/useRef

```js
function MyInput({ value, onChange, ref }) {
  return <input value={value} onChange={onChange} ref={ref} />;
}
```

https://react.dev/reference/react/useRef

- `useImperativeHandle(ref, createHandle, deps?)` takes the `ref` prop. The real `<input>` then needs its own inner `useRef`. Omitting `deps` recreates the handle every render. https://react.dev/reference/react/useImperativeHandle
- Use a handle only for what a prop cannot express (focus, scroll, animation, selection). Prefer `<Modal isOpen={isOpen} />` to `{ open, close }`. https://react.dev/reference/react/useImperativeHandle
- Expose DOM nodes from low-level components (buttons, inputs), not from app-level ones. https://react.dev/reference/react/forwardRef
- Refs passed to class components are not props: they reference the instance. https://react.dev/blog/2024/12/05/react-19

## Context

- Render the context itself as the provider: `<ThemeContext value={theme}>`. https://react.dev/reference/react/createContext

```js
export function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
  return (
    <TasksContext value={tasks}>
      <TasksDispatchContext value={dispatch}>{children}</TasksDispatchContext>
    </TasksContext>
  );
}
```

https://react.dev/learn/scaling-up-with-reducer-and-context

- The default value is used only when no provider exists above. A provider without `value`, or with a misspelled prop (`theme=`), provides `undefined`. https://react.dev/reference/react/useContext
- `useContext` reads the closest provider above the caller. A provider rendered by the same component does not count. https://react.dev/reference/react/useContext
- Every consumer re-renders when `value` changes by `Object.is`, and `memo` does not stop it. An inline `{ currentUser, login }` is new each render: wrap the function in `useCallback` and the object in `useMemo` when it matters. https://react.dev/reference/react/useContext
- To react to part of a context, read it in an outer component and pass the piece to a memoized child. https://react.dev/reference/react/memo
- Two copies of the context object (build tooling, symlinks) give two different contexts. Assign each to a global and compare them with `===`. https://react.dev/reference/react/useContext
- Before reaching for context, pass props, or pass JSX as `children` to skip layers. https://react.dev/learn/passing-data-deeply-with-context
- Reducer plus context: one context for state, one for `dispatch`, a provider component and custom Hooks `useTasks()` and `useTasksDispatch()` in one file. https://react.dev/learn/scaling-up-with-reducer-and-context
- TypeScript, no sensible default: `createContext<T | null>(null)` and a consumer Hook that throws "must be used within a Provider". https://react.dev/learn/typescript
- `use(context)` reads a context like `useContext` but may be called conditionally or after an early return. https://react.dev/blog/2024/12/05/react-19

## useId

- One `useId()` per component as a prefix for several elements: `id + '-firstName'`. https://react.dev/reference/react/useId
- Never for list keys, nor as a cache key for `use()`. Keys come from data. https://react.dev/reference/react/useId
- IDs derive from the parent path, so server and client trees must match. Several React roots on one page: give each root its own `identifierPrefix` in `createRoot` or `hydrateRoot`; a server-rendered root uses the same prefix in the server API. https://react.dev/reference/react/useId
- Not usable in async Server Components. https://react.dev/reference/react/useId
- The default prefix is `_r_`, valid in `view-transition-name` and XML names. https://react.dev/blog/2025/10/01/react-19-2

## useSyncExternalStore

- For non-React stores and browser APIs. Prefer `useState` or `useReducer` for your own state. https://react.dev/reference/react/useSyncExternalStore
- `getSnapshot` must return the same value while the store is unchanged. `return { todos: store.todos }` loops with "The result of getSnapshot should be cached". https://react.dev/reference/react/useSyncExternalStore
- Define `subscribe` outside the component, otherwise React resubscribes every render. https://react.dev/reference/react/useSyncExternalStore
- `getServerSnapshot` must return the same data on the server and the first client render. Omitting it makes server rendering of the component throw. https://react.dev/reference/react/useSyncExternalStore
- Store updates cannot be Transitions: a change during a Transition falls back to a blocking update. https://react.dev/reference/react/useSyncExternalStore

```js
useSyncExternalStore(subscribe, () => navigator.onLine, () => true);
```

https://react.dev/learn/you-might-not-need-an-effect

## useDebugValue

- Most valuable in custom Hooks of shared libraries with a complex internal structure. Pass a formatter to defer cost: `useDebugValue(date, d => d.toDateString())`. https://react.dev/reference/react/useDebugValue
