# Actions: useActionState, useOptimistic, useFormStatus, form actions

## Contents

- useActionState
- Errors, reset, cancellation
- useOptimistic
- useFormStatus
- form action

## useActionState

`const [state, dispatchAction, isPending] = useActionState(reducerAction, initialState, permalink?)` https://react.dev/reference/react/useActionState

- The reducer gets `(previousState, payload)`. With a `<form>`, `FormData` is the second argument, not the first. https://react.dev/reference/react/useActionState

```js
async function action(prevState, formData) {
  const name = formData.get('name');
}
```

https://react.dev/reference/react/useActionState

- Unlike `useReducer`, the reducer may be async with side effects and is not double-invoked in Strict Mode. https://react.dev/reference/react/useActionState
- Call `dispatchAction` inside a Transition: pass it to `<form action>`, `formAction` or an `xxxAction` prop, or wrap it in `startTransition`. Outside one, `isPending` never updates and development logs "An async function with useActionState was called outside of a transition". https://react.dev/reference/react/useActionState
- Calling it during render throws "Cannot update action state while rendering". https://react.dev/reference/react/useActionState
- Calls are queued and run one after another, each receiving the previous result. For parallel work use `useState` with `useTransition`. https://react.dev/reference/react/useActionState
- `initialState` is ignored after the first dispatch. The reducer's return type must match it. https://react.dev/reference/react/useActionState
- State set after an `await` inside the reducer needs its own `startTransition`. https://react.dev/reference/react/useActionState
- `dispatchAction` is stable. With Server Functions, `initialState` and the payload must be serializable. https://react.dev/reference/react/useActionState

## Errors, reset, cancellation

- Return known errors (validation) as state. A thrown error cancels every queued action and reaches the nearest Error Boundary. https://react.dev/reference/react/useActionState

```js
async function myReducerAction(prevState, data) {
  try {
    return { success: true, data: await submitData(data) };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

https://react.dev/reference/react/useActionState

- There is no built-in reset. Handle a reset payload (`dispatchAction(null)` returning `initialState`), change a `key`, or rely on `<form action>`, which resets uncontrolled fields on success. https://react.dev/reference/react/useActionState
- Cancel by passing an `AbortController` signal in the payload and aborting the previous one. An abort never undoes a server mutation, so useActionState does not abort by default. https://react.dev/reference/react/useActionState
- `permalink`: with a Server Function, a submission before hydration navigates there. Render the same form, reducer and permalink on that page. https://react.dev/reference/react/useActionState

## useOptimistic

- `useOptimistic(value, reducer?)` returns `[optimisticState, setOptimistic]`. `value` shows when no Action is pending. https://react.dev/reference/react/useOptimistic
- Call the setter inside an Action. Outside, the value flashes and reverts with "An optimistic state update occurred outside a Transition or Action". During render it throws. https://react.dev/reference/react/useOptimistic
- Inside an Action prop no `startTransition` is needed for the setter. https://react.dev/reference/react/useOptimistic

```js
async function submitAction(formData) {
  const newName = formData.get('name');
  setOptimisticName(newName);

  const updatedName = await updateName(newName);
  startTransition(() => {
    action(updatedName);
  });
}
```

https://react.dev/reference/react/useOptimistic

- When the Action ends, the real `value` shows in the same commit. On failure the UI reverts: catch to show a message. https://react.dev/reference/react/useOptimistic
- Prefer a relative reducer, `(current, delta) => current + delta`, to absolute values: React re-runs the reducer if the base changes while pending. https://react.dev/reference/react/useOptimistic
- Pending: `optimistic !== value`, `useTransition`, or an `isPending: true` flag per item. https://react.dev/reference/react/useOptimistic
- With useActionState, call `setOptimistic` and `dispatchAction` in the same `startTransition`. https://react.dev/reference/react/useActionState

## useFormStatus

- Import from `react-dom`. `const { pending, data, method, action } = useFormStatus();`. https://react.dev/reference/react-dom/hooks/useFormStatus
- It reads the parent `<form>` only. A form rendered by the same component is not tracked, so `pending` stays false. Extract a child. https://react.dev/reference/react-dom/hooks/useFormStatus

```js
function Submit() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}
```

https://react.dev/reference/react-dom/hooks/useFormStatus

- `action` is `null` when the form's `action` is a URL. https://react.dev/reference/react-dom/hooks/useFormStatus

## form action

- `action` takes a URL or a function. A function runs in a Transition, may be async, and gets one argument, the `FormData`. `formAction` on a submit button overrides it. https://react.dev/reference/react-dom/components/form
- With a function the method is always POST. `e.preventDefault()` is not needed. After success, every uncontrolled field resets. `requestFormReset` resets manually. https://react.dev/reference/react-dom/components/form and https://react.dev/blog/2024/12/05/react-19
- Errors thrown by `action` reach the nearest Error Boundary. https://react.dev/reference/react-dom/components/form
- Extra data: hidden inputs, or `.bind`, whose bound arguments come before `formData`. https://react.dev/reference/react-dom/components/form

```jsx
async function addToCart(productId, formData) {
  "use server";
  await updateCart(productId);
}
<form action={addToCart.bind(null, productId)}>
```

https://react.dev/reference/react-dom/components/form

- One form, several submissions: a different `formAction` per button. https://react.dev/reference/react-dom/components/form
- A Server Function passed to `<form action>` lets users submit before the JavaScript has loaded. Showing an error before hydration also needs the form in a Client Component with `useActionState`. https://react.dev/reference/react-dom/components/form
- `onSubmit` still works but gives no transition tracking, no Error Boundary routing, no Server Functions and no useActionState or useOptimistic integration. https://react.dev/reference/react-dom/components/form
- `onReset` fires after a Server Action form reset, and submit events include `submitter`. https://react.dev/blog/2026/09/09/react-19-3
