# Memoization: useMemo, useCallback, memo

## Contents

- With React Compiler
- When manual memoization pays
- useMemo
- useCallback
- memo
- Needing less memoization

## With React Compiler

- The compiler memoizes values, functions and JSX automatically. With it you typically do not need `memo`, `useMemo` or `useCallback`. https://react.dev/reference/react/memo and https://react.dev/learn/react-compiler/introduction
- New code: rely on the compiler and keep `useMemo` or `useCallback` for precise control, typically a value used as an Effect dependency. https://react.dev/learn/react-compiler/introduction
- Existing code: leave manual memoization in place, or remove it only with tests. The compiler keeps it and compiles a component only when its inference matches or exceeds it. https://react.dev/learn/react-compiler/introduction and https://react.dev/reference/eslint-plugin-react-hooks/lints/preserve-manual-memoization
- Incomplete dependencies of `useMemo` or `useCallback` prevent the compiler from understanding the data flow and applying further optimizations (lint `preserve-manual-memoization`). Complete them or drop the manual memoization. https://react.dev/reference/eslint-plugin-react-hooks/lints/preserve-manual-memoization

## When manual memoization pays

- `useMemo` only for: a noticeably slow calculation with rarely changing dependencies, a prop of a `memo` component, a dependency of another Hook. https://react.dev/reference/react/useMemo
- `useCallback` only for: a prop of a `memo` component, a dependency of another Hook. https://react.dev/reference/react/useCallback
- Measure first: `console.time` around the calculation. About 1 ms or more per interaction may justify it. Test with CPU throttling in a production build. https://react.dev/reference/react/useMemo
- `useMemo` does not speed up the first render. https://react.dev/reference/react/useMemo
- `memo` pays when a component re-renders often with identical props and rendering is expensive. https://react.dev/reference/react/memo

## useMemo

- Use it only as a performance optimization. The cache is dropped when the file is edited in development and when the component suspends during its initial mount. If the code breaks without it, fix the code. https://react.dev/reference/react/useMemo
- The calculation is pure and takes no arguments. Strict Mode calls it twice in development. https://react.dev/reference/react/useMemo
- A brace after the arrow opens a body: `useMemo(() => { matchMode: 'whole-word', text: text }, [text])` does not return an object. Write `return { ... }`. https://react.dev/reference/react/useMemo
- No dependency array recalculates every render. https://react.dev/reference/react/useMemo
- Build objects inside the callback and depend on primitives. https://react.dev/reference/react/useMemo

```js
const visibleItems = useMemo(() => {
  const searchOptions = { matchMode: 'whole-word', text };
  return searchItems(allItems, searchOptions);
}, [allItems, text]);
```

https://react.dev/reference/react/useMemo

- An object used as an Effect dependency: move it inside the Effect rather than memoizing it. https://react.dev/reference/react/useMemo
- In a list, extract an item component that calls `useMemo`, or wrap the item in `memo`. https://react.dev/reference/react/useMemo
- `useMemo` must return a value. Without one it returns `undefined`, so side effects belong in event handlers or `useEffect` (lint `use-memo`). https://react.dev/reference/eslint-plugin-react-hooks/lints/use-memo
- In Server Components, `cache` memoizes a data fetch or computation and shares it between components for one server request. https://react.dev/reference/react/cache

## useCallback

- `useCallback(fn, deps)` is `useMemo(() => fn, deps)`. It does not avoid creating the function. https://react.dev/reference/react/useCallback
- Keep dependencies few with an updater. https://react.dev/reference/react/useCallback

```js
const handleAddTodo = useCallback((text) => {
  const newTodo = { id: nextId++, text };
  setTodos(todos => [...todos, newTodo]);
}, []);
```

https://react.dev/reference/react/useCallback

- A function listed as an Effect dependency: move it inside the Effect rather than wrapping it. https://react.dev/reference/react/useCallback
- `onClick={() => handleClick(item)}` defeats a manual `useCallback`. The compiler handles both forms. https://react.dev/learn/react-compiler/introduction
- TypeScript strict mode needs typed parameters: `useCallback<React.ChangeEventHandler<HTMLInputElement>>((event) => ..., [setValue])`. https://react.dev/learn/typescript

## memo

- `memo` is a hint, not a guarantee: React may still re-render. https://react.dev/reference/react/memo
- Props are compared one by one with `Object.is`. An object, array or function created in the parent each render defeats it. https://react.dev/reference/react/memo
- A memoized component still re-renders on its own state change and on context changes. https://react.dev/reference/react/memo
- Reduce prop changes, in this order: pass primitives, pass a coarser value (`hasGroups` rather than `person`), `useMemo` the object, declare the function outside or `useCallback` it. https://react.dev/reference/react/memo
- A custom `arePropsEqual` must compare every prop, functions included, or handlers see stale values. Avoid deep equality. https://react.dev/reference/react/memo
- Setting state to its current value skips the re-render without `memo`. https://react.dev/reference/react/memo

## Needing less memoization

1. Accept JSX as `children`, so a wrapper's state change does not re-render them.
2. Keep state local.
3. Keep rendering pure: a visual glitch on re-render is a bug, not a reason to memoize.
4. Remove Effects that update state: Effect chains cause most performance problems.
5. Remove needless Effect dependencies.

https://react.dev/reference/react/useMemo

- Put a text input's state in its own component so heavy siblings do not re-render on each keystroke. https://react.dev/reference/react-dom/components/input
- Creating JSX elements is cheap: do not optimize it. https://react.dev/reference/react/createElement
