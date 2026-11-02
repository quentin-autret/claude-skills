# Upgrading to React 19.x

## Contents

- Steps
- Removed APIs
- Deprecated APIs
- Behavior changes
- TypeScript changes
- Changes in 19.2 and 19.3

## Steps

1. Upgrade to `react@18.3` first: it is 18.2 plus warnings for APIs removed in 19.
2. Install with exact versions: `npm install --save-exact react@^19.0.0 react-dom@^19.0.0`, and `@types/react@^19.0.0 @types/react-dom@^19.0.0`.
3. The new JSX transform is required ("outdated JSX transform" warning).
4. Run `npx codemod@latest react/19/migration-recipe` and `npx types-react-codemod@latest preset-19 ./path-to-app`.

https://react.dev/blog/2024/04/25/react-19-upgrade-guide

- Libraries reading `SECRET_INTERNALS...` block the upgrade: remove code that touches React internals. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- UMD builds are gone: load from an ESM CDN, `import React from "https://esm.sh/react@19/?dev"`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide

## Removed APIs

| Removed | Replacement |
| ------- | ----------- |
| `propTypes` (silently ignored) | TypeScript |
| `defaultProps` on function components | ES default parameters (classes keep `defaultProps`) |
| `contextTypes`, `childContextTypes`, `getChildContext` | `createContext` and `static contextType` |
| String refs, `this.refs` | Ref callbacks or `createRef` |
| Module pattern factories | Regular components |
| `React.createFactory` | JSX |
| `react-test-renderer/shallow` | `react-shallow-renderer`, better Testing Library |
| `react-dom/test-utils` | `act` from `react`, other utilities throw |
| `ReactDOM.render` | `createRoot(el).render()` |
| `ReactDOM.hydrate` | `hydrateRoot` |
| `unmountComponentAtNode` | `root.unmount()` |
| `ReactDOM.findDOMNode` | DOM refs |
| `renderToNodeStream`, `renderToStaticNodeStream` | Current `react-dom/server` APIs |

https://react.dev/blog/2024/04/25/react-19-upgrade-guide and https://react.dev/reference/react-dom

- Also removed: `unstable_flushControlled`, `unstable_createEventHandle`, `unstable_renderSubtreeIntoContainer`, `unstable_runWithPriority`, `errorInfo.digest` in `onRecoverableError`, deprecated `react-is` methods. https://react.dev/blog/2024/04/25/react-19-upgrade-guide

## Deprecated APIs

- `element.ref`: read `element.props.ref`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- `react-test-renderer`: migrate to Testing Library. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- `forwardRef`: no longer necessary, pass `ref` as a prop (it will be deprecated in a future release). https://react.dev/reference/react/forwardRef
- `<Context.Provider>`: render `<Context value>` (deprecation planned for future versions). https://react.dev/blog/2024/12/05/react-19
- `ReactDOM.useFormState`: renamed `useActionState` from `react`. https://react.dev/blog/2024/12/05/react-19
- `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`: only the `UNSAFE_` names will keep working (codemod `rename-unsafe-lifecycles`). https://react.dev/reference/react/Component
- `setState` inside `componentDidCatch`. https://react.dev/reference/react/Component

## Behavior changes

- Render errors are not re-thrown: uncaught go to `window.reportError`, caught to `console.error`. Use root options `onUncaughtError`, `onCaughtError`, `onRecoverableError`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Suspense commits the nearest fallback immediately, then pre-warms suspended siblings. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Strict Mode: `useMemo` and `useCallback` reuse the first result on the second render, ref callbacks double-run on mount. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Transitions in `popstate` are synchronous. `javascript:` URLs in `src` and `href` are errors. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Hydration mismatch logs a single diff. https://react.dev/blog/2024/12/05/react-19
- A ref callback may return a cleanup function. React then skips calling it with `null` on unmount. https://react.dev/blog/2024/12/05/react-19

## TypeScript changes

- A ref callback must not implicitly return: `ref={current => {instance = current}}`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- `useRef` needs an argument: `useRef(undefined)`, `useRef<T>(null)`. All refs are mutable, `MutableRef` is deprecated. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- `ReactElement["props"]` is `unknown`, not `any` (extra codemod `react-element-default-any-props`). https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- The global `JSX` namespace is gone. Augment it in `react/jsx-runtime` for `jsx: react-jsx`, `react/jsx-dev-runtime` for `react-jsxdev`, `react` for `react` or `preserve`. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- `useReducer`: no type arguments, or `useReducer<State, [Action]>(reducer)`. Annotate inline reducer parameters. https://react.dev/blog/2024/04/25/react-19-upgrade-guide

## Changes in 19.2 and 19.3

- 19.2 adds `<Activity>`, `useEffectEvent`, `cacheSignal`, Performance tracks and partial prerendering (`prerender` with `signal`, then `resume`). https://react.dev/blog/2025/10/01/react-19-2
- 19.2: SSR batches Suspense reveals like the client. The `useId` prefix becomes `_r_`. ARIA 1.3 attributes no longer warn. https://react.dev/blog/2025/10/01/react-19-2
- 19.3 makes `<ViewTransition>` and `addTransitionType` stable and adds Fragment refs and `use(browser())`. https://react.dev/blog/2026/09/09/react-19-3
- 19.3: Trusted Types values pass without string coercion. Server Components render a client context directly. Unrelated Transitions render independently. Strict Mode double-invokes Effects during hydration. https://react.dev/blog/2026/09/09/react-19-3
- 19.3 fixes: `useDeferredValue` stuck on an old value, `useEffectEvent` stale inside `forwardRef` or `memo`, `useSyncExternalStore` missing mutations while an Activity is hidden, number input `defaultValue`, false `nonce` hydration mismatches. https://react.dev/blog/2026/09/09/react-19-3
- Older majors have their own sites: 18.react.dev, 17.react.dev. https://react.dev/versions
