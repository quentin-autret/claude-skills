# JSX

## Contents

- Setup
- Where the `JSX` namespace comes from
- Type checking rules

## Setup

- Name files `.tsx` and set `jsx`. `react-jsx` emits the automatic runtime (`_jsx`), `react-jsxdev` the dev runtime (`_jsxDEV`), `react` the classic `React.createElement`, `preserve` keeps JSX in `.jsx`, `react-native` keeps JSX in `.js`. https://www.typescriptlang.org/docs/handbook/jsx.html
- `jsxFactory`, `jsxFragmentFactory` and `jsxImportSource` work as options and as per-file pragmas. `jsxFactory` sets the factory for the classic modes (default `React.createElement`). https://www.typescriptlang.org/docs/handbook/jsx.html
- `<Foo>bar` assertions are not allowed in `.tsx`: write `bar as Foo`. https://www.typescriptlang.org/docs/handbook/jsx.html
- Under `moduleDetection: auto`, a JSX file under `react-jsx` is a module. https://www.typescriptlang.org/tsconfig/#moduleDetection

## Where the `JSX` namespace comes from

- Classic modes: on the factory's top-level identifier (`React.JSX`, `h.JSX`). `react-jsx`: exported by `${jsxImportSource}/jsx-runtime`. `react-jsxdev`: by `${jsxImportSource}/jsx-dev-runtime`. Otherwise the global `JSX`. https://www.typescriptlang.org/docs/handbook/jsx.html
- A library exposing a runtime maps the extensionless subpaths in `exports`, and exports `jsx`, `jsxs`, `Fragment` (production) and `jsxDEV`, `Fragment` (dev) with types. https://www.typescriptlang.org/docs/handbook/jsx.html

```json
{ "exports": { "./jsx-runtime": "./jsx-runtime.js", "./jsx-dev-runtime": "./jsx-dev-runtime.js" } }
```

https://www.typescriptlang.org/docs/handbook/jsx.html

## Type checking rules

- A lowercase tag is intrinsic (`JSX.IntrinsicElements`), an uppercase tag is a value in scope. Without a declared `JSX.IntrinsicElements`, intrinsic elements are not checked at all. https://www.typescriptlang.org/docs/handbook/jsx.html
- `JSX.ElementType` declares what may be a tag, so a library can accept components that return a `Promise` or a `string` (since 5.1). Props still come from the first parameter. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html#decoupled-type-checking-between-jsx-elements-and-jsx-tag-types
- A function component returns something assignable to `JSX.Element | null` by default. A class instance must match `JSX.ElementClass`. https://www.typescriptlang.org/docs/handbook/jsx.html
- Attribute names that are not identifiers (`data-*`) never error when absent from the props type. Spread attributes are checked for required props. https://www.typescriptlang.org/docs/handbook/jsx.html
- `JSX.IntrinsicAttributes` holds framework attributes such as `key`. `JSX.IntrinsicClassAttributes<T>` applies to class components. Keep their properties optional. https://www.typescriptlang.org/docs/handbook/jsx.html
- A `children: JSX.Element` prop rejects several children and text children. https://www.typescriptlang.org/docs/handbook/jsx.html
- A JSX expression is `any` by default, and `JSX.Element` is opaque: element, props and children types cannot be recovered from it. https://www.typescriptlang.org/docs/handbook/jsx.html
- Namespaced names work: `<Foo a:b="hello" />` uses the key `"a:b"`, and a namespaced tag whose first segment is lowercase is looked up as `JSX.IntrinsicElements["a:b"]` (since 5.1). https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html#namespaced-jsx-attributes
- `SFC` and `StatelessComponent` are deprecated: type plain function components. https://www.typescriptlang.org/docs/handbook/jsx.html
