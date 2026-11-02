# Server Functions and 'use server'

## Contents

- Terms
- Writing 'use server'
- Security
- Calling a Server Function
- Arguments and return values

## Terms

- A Server Function runs on the server and is callable from the client. One passed to an action prop or called inside an Action is a Server Action. The older name "Server Actions" covered every Server Function. https://react.dev/reference/rsc/server-functions
- Stable across 19.x minors, but bundler and framework APIs that implement them may break between minors. Framework authors pin an exact React version. https://react.dev/reference/rsc/server-functions

## Writing 'use server'

- First statement of an async function or of the module, above imports. Single or double quotes, never backticks. https://react.dev/reference/rsc/use-server
- Only on async functions, only in server files. A file-level `'use server'` makes every export a Server Function. https://react.dev/reference/rsc/use-server
- Client code imports Server Functions only from a module with a file-level `'use server'`. Inline `'use server'` works inside a Server Component, which passes the function as a prop. https://react.dev/reference/rsc/server-functions and https://react.dev/reference/rsc/use-server
- Meant for mutations, not data fetching: frameworks usually run one at a time and cannot cache the result. https://react.dev/reference/rsc/use-server

## Security

- Arguments are fully client-controlled. Validate and escape them, and check in every Server Function that the user may perform the action. https://react.dev/reference/rsc/use-server
- Abstract secrets into a helper file tagged with `import "server-only"`, so the client cannot import it. https://react.dev/reference/react/experimental_taintUniqueValue

## Calling a Server Function

- From `<form action>` or `formAction`, React wraps the call in a Transition and, on success, resets the uncontrolled fields of the form. https://react.dev/reference/react-dom/components/form
- With `useActionState`, React replays submissions made before hydration. The `permalink` third argument redirects a submission made before the bundle loads. https://react.dev/reference/rsc/server-functions

```js
const [, submitAction] = useActionState(updateName, null, `/name/update`);
```

https://react.dev/reference/rsc/server-functions

- `useActionState`, like most Hooks, can only be called in client code. https://react.dev/reference/rsc/use-server
- Outside a form, call it inside `startTransition` for `isPending`, then set state in a nested `startTransition` after the `await`. https://react.dev/reference/rsc/server-functions

```js
startTransition(async () => {
  const { error } = await updateName(name);
  startTransition(() => {
    if (error) setError(error);
    else setName('');
  });
});
```

https://react.dev/reference/rsc/server-functions

- The root's `formState` option carries the result of a permalink submission into `hydrateRoot`. It must equal the value given to the server renderer, and frameworks usually handle it. https://react.dev/reference/react-dom/client/hydrateRoot

## Arguments and return values

- Allowed arguments: primitives, `Symbol.for` symbols, String, Array, Map, Set, TypedArray, ArrayBuffer, Date, FormData, plain objects, Server Functions, Promises. https://react.dev/reference/rsc/use-server
- Not allowed: JSX, plain functions, classes and instances, null-prototype objects, non-global symbols, event objects. https://react.dev/reference/rsc/use-server
- Return values follow the rules of props passed to a Client Component. https://react.dev/reference/rsc/use-server
- Bound arguments come before `formData`: `addToCart.bind(null, productId)` calls `addToCart(productId, formData)`. https://react.dev/reference/react-dom/components/form
