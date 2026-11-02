# Legacy APIs: Children, cloneElement, class components, forwardRef

## Contents

- Status
- Children
- cloneElement and isValidElement
- forwardRef and createRef
- Class components
- PureComponent

## Status

- Legacy, not for new code: `Children`, `cloneElement`, `Component`, `createElement`, `createRef`, `forwardRef`, `isValidElement`, `PureComponent`. https://react.dev/reference/react/legacy

## Children

- `Children` leads to fragile code. The lowercase `children` prop is the normal way. https://react.dev/reference/react/Children
- `children` is opaque: never assume it is an array, since a single child is not wrapped in an array. https://react.dev/reference/react/Children
- Traversal stops at elements: `<MoreRows />` counts as 1 child, whatever it renders. Fragments are not traversed. https://react.dev/reference/react/Children
- `Children.only` throws on any array, even of 1 element. `Children.toArray` drops `null`, `undefined` and booleans. https://react.dev/reference/react/Children
- Prefer: an exported `Row` component the caller wraps, an array prop, or a render prop. https://react.dev/reference/react/Children

```js
<RowList rows={[{ id: 'first', content: <p>First</p> }]} />
```

https://react.dev/reference/react/Children

## cloneElement and isValidElement

- `cloneElement(element, props, ...children)` merges shallowly, props win, `key` and `ref` replace the originals. https://react.dev/reference/react/cloneElement
- Prefer a render prop (`renderItem(item, isHighlighted)`), a context, or a custom Hook. https://react.dev/reference/react/cloneElement
- `isValidElement` is true only for elements. It is not a "can render" check: `42` renders but is not an element. https://react.dev/reference/react/isValidElement

## forwardRef and createRef

- In React 19, `forwardRef` is no longer necessary: pass `ref` as a prop. It will be deprecated in a future release. https://react.dev/reference/react/forwardRef
- `ref` is `null` when the component never uses it or renders the node conditionally. https://react.dev/reference/react/forwardRef
- `createRef` returns a new object each call and suits classes. Function components use `useRef(null)`. https://react.dev/reference/react/createRef

## Class components

- `setState` does not change `this.state` immediately: read the new value in the `callback` argument or `componentDidUpdate`. Pass a function to base it on the previous state. https://react.dev/reference/react/Component
- In development, Strict Mode calls `constructor` and `render` twice and mounts, unmounts, mounts again. The constructor runs on the server too. Keep side effects and subscriptions out of it. https://react.dev/reference/react/Component
- `setState` in `componentDidMount` adds a render before paint: only to measure the DOM. https://react.dev/reference/react/Component
- `componentDidUpdate` compares `prevProps` or `prevState` before setting state, or it loops. https://react.dev/reference/react/Component
- `shouldComponentUpdate` is a hint only. No deep equality, no `JSON.stringify`. Prefer `PureComponent`. https://react.dev/reference/react/Component
- `UNSAFE_componentWillMount` may never mount under Suspense: subscribe in `componentDidMount`. No `UNSAFE_` method runs when `getDerivedStateFromProps` or `getSnapshotBeforeUpdate` exists. https://react.dev/reference/react/Component
- `getDerivedStateFromProps` runs on every render. `getDerivedStateFromError` is pure, side effects go in `componentDidCatch`. https://react.dev/reference/react/Component
- `static defaultProps` applies to `undefined` and missing props, not `null`. `static contextType` gives `this.context`. https://react.dev/reference/react/Component
- In development, errors caught by `componentDidCatch` still reach `window.onerror`. In production they do not. https://react.dev/reference/react/Component

## PureComponent

- A class with shallow `shouldComponentUpdate`. It still re-renders on context change. https://react.dev/reference/react/PureComponent
- Migrate with `memo`. `memo` does not compare state, since setting identical state already bails out. https://react.dev/reference/react/PureComponent
