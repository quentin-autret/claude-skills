# State: useState, useReducer, state structure

## Contents

- useState traps
- Updater functions and the update queue
- Setting state during render
- useReducer
- Objects and arrays
- State structure
- Preserving and resetting state

## useState traps

- Pass the initializer itself, not its result: `useState(createInitialTodos)` runs once, `useState(createInitialTodos())` runs on every render and the result is thrown away. The initializer must be pure and take no arguments. https://react.dev/reference/react/useState
- To store a function in state, wrap it: `useState(() => fn)` and `setFn(() => otherFn)`. Otherwise React calls it as an initializer or updater. https://react.dev/reference/react/useState
- `set` skips the re-render (children included) when the next value is `Object.is`-equal. Mutating an object then calling `setObj(obj)` does nothing. https://react.dev/reference/react/useState
- `set` only affects the next render. Reading the variable right after, even in a `setTimeout`, gives the old value. Keep the new value in a local: `const nextCount = count + 1; setCount(nextCount);`. https://react.dev/reference/react/useState
- A value never changes inside one render, even in async handlers: handlers see the state of the render that created them. https://react.dev/learn/state-as-a-snapshot
- `set` has a stable identity. Omitting it from Effect dependencies is fine. https://react.dev/reference/react/useState
- "Too many re-renders" usually means `onClick={handleClick()}` instead of `onClick={handleClick}`. https://react.dev/reference/react/useState
- Do not mirror a prop in state (`useState(messageColor)` ignores later changes). Mirror only to ignore updates on purpose, and name the prop `initialX` or `defaultX`. https://react.dev/learn/choosing-the-state-structure

## Updater functions and the update queue

- An updater is queued, any other value queues "replace with X": `setNumber(number + 5); setNumber(n => n + 1); setNumber(42)` ends at 42. Three `setAge(age + 1)` give 43, three `setAge(a => a + 1)` give 45. https://react.dev/learn/queueing-a-series-of-state-updates and https://react.dev/reference/react/useState
- An updater is not always required: React guarantees a click sees the state updated by the previous click. Use one for several updates in the same event, or when async callbacks may read a stale value (`setPending(p => p + 1)`). https://react.dev/reference/react/useState and https://react.dev/learn/queueing-a-series-of-state-updates
- Updaters run during render: pure, no `set` calls, no side effects. Strict Mode calls them twice in development, so `prevTodos.push(...)` shows a double add. https://react.dev/reference/react/useState
- React does not batch across separate intentional events: a first click that disables a form protects against a second submit. https://react.dev/learn/queueing-a-series-of-state-updates

## Setting state during render

- Allowed only for the component being rendered, behind a guard, otherwise it loops. Setting another component's state during render is an error. https://react.dev/reference/react/useState

```js
if (prevItems !== items) {
  setPrevItems(items);
  setSelection(null);
}
```

https://react.dev/learn/you-might-not-need-an-effect

- Try first, in order: derive the value during render, reset the subtree with a `key`, update everything in the event handler. The guarded pattern still beats an Effect, because children do not render twice. https://react.dev/reference/react/useState
- Clamp where state is set, not in render: `setCount(c => Math.min(c + 1, max))`. `if (count > max) setCount(max)` in render loops. https://react.dev/reference/eslint-plugin-react-hooks/lints/set-state-in-render

## useReducer

- `useReducer(reducer, initialArg, init?)`: pass `init` to avoid recomputing, `useReducer(reducer, username, createInitialState)`, not `createInitialState(username)`. https://react.dev/reference/react/useReducer
- The reducer is pure and returns new objects. `state.age++; return state;` never re-renders (same object). https://react.dev/reference/react/useReducer
- `dispatch` only affects the next render. To see the result now, call `reducer(state, action)` yourself. https://react.dev/reference/react/useReducer
- Part of the state becomes `undefined`: a case forgot `...state`. All of it: a case forgot to `return`, or no case matched. End the `switch` with `throw Error('Unknown action: ' + action.type)`. https://react.dev/reference/react/useReducer
- One action per user interaction, minimal payload, names like `incremented_age`. https://react.dev/reference/react/useReducer
- TypeScript: type the initial state and the reducer signature `(state: State, action: Action): State` rather than passing type arguments. https://react.dev/learn/typescript

## Objects and arrays

- Spread is one level deep. A nested update copies every level: `setPerson({...person, artwork: {...person.artwork, city: 'New Delhi'}})`. https://react.dev/learn/updating-objects-in-state
- Nested objects are shared references: mutating one can move an unrelated prop that points to the same object. `+=` on a state field is a mutation. https://react.dev/learn/updating-objects-in-state
- Mutating an object you just created, not yet in state, is fine, even during render. https://react.dev/learn/updating-objects-in-state
- Arrays: add with `[...arr, x]` or `concat`, remove with `filter`, replace with `map`, insert with `[...arr.slice(0, i), x, ...arr.slice(i)]`. `slice` copies, `splice` mutates. https://react.dev/learn/updating-arrays-in-state
- A copied array still shares its items: `nextList[0].seen = true` after `const nextList = [...list]` mutates the original. Update an item with `map` and a new object. https://react.dev/learn/updating-arrays-in-state
- `setItems(items.sort())` mutates state, because `sort` mutates. Copy first: `const nextList = [...list]; nextList.sort();`. https://react.dev/reference/eslint-plugin-react-hooks/lints/immutability and https://react.dev/learn/updating-arrays-in-state
- For deep updates, flatten the state or use Immer (`useImmer`, `useImmerReducer`), where `push` on the draft is fine. https://react.dev/learn/updating-arrays-in-state

## State structure

- Replace contradicting booleans (`isSending`, `isSent`) by one `status` and derive the booleans as constants. https://react.dev/learn/choosing-the-state-structure
- Grouped state: `setPosition({x: 100})` drops `y`. Write `{...position, x: 100}`. https://react.dev/learn/choosing-the-state-structure
- Store `selectedId`, not a copy of the selected object, so edits to `items` propagate. https://react.dev/learn/choosing-the-state-structure
- Normalize deep trees into a table keyed by id with `childIds`. https://react.dev/learn/choosing-the-state-structure
- TypeScript: model request state as a discriminated union on `status`. https://react.dev/learn/typescript

## Preserving and resetting state

- State belongs to a position in the render tree. Same type at the same position keeps state. Two `<Counter />` in the two branches of a ternary share state. Changing a wrapper from `section` to `div` resets everything below. https://react.dev/learn/preserving-and-resetting-state
- Never define a component inside another: each render makes a new type, so state resets. https://react.dev/learn/preserving-and-resetting-state
- Reset with a `key` on any component, not only in lists: `<Chat key={to.id} contact={to} />`. Keys are scoped to the parent. https://react.dev/learn/preserving-and-resetting-state
- Fragment nesting: `<><Child /></>` to `<Child />` keeps state, `<><><Child /></></>` to `<Child />` resets it. https://react.dev/reference/react/Fragment
