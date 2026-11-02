# Rules of React, eslint-plugin-react-hooks and Strict Mode

## Contents

- Purity
- React calls components and Hooks
- Rules of Hooks
- eslint-plugin-react-hooks
- Lint rules in brief
- Strict Mode

## Purity

- The rules are rules, not guidelines: "if they are broken, your app likely has bugs". Enforce them with Strict Mode and `eslint-plugin-react-hooks`. https://react.dev/reference/rules
- Render runs many times. `new Date()`, `Math.random()`, `crypto.randomUUID()`, `performance.now()` in render are bugs (lint `purity`). Keep a stable id with `useState(() => crypto.randomUUID())`, a clock with `useState(() => Date.now())` plus `setInterval` in an Effect. https://react.dev/reference/eslint-plugin-react-hooks/lints/purity
- Top-level component code runs in render. Code in handlers and Effects does not. Prefer handlers for side effects, `useEffect` as a last resort. https://react.dev/reference/rules/components-and-hooks-must-be-pure
- Local mutation of something created in render is fine. Pushing into a module-level array duplicates items each render. https://react.dev/reference/rules/components-and-hooks-must-be-pure
- No global writes in render (`window.x = ...`, module counters, module caches, lint `globals`). Use state, context, or an Effect for `document.title = title`. https://react.dev/reference/eslint-plugin-react-hooks/lints/globals
- Props, state, context, Hook arguments and Hook return values are immutable. Mutating a Hook argument returns a stale memoized result. https://react.dev/reference/rules/components-and-hooks-must-be-pure
- Values passed to JSX are frozen: mutating `styles` after `<Header styles={styles} />` gives stale UI. https://react.dev/reference/rules/components-and-hooks-must-be-pure
- Components must not coordinate through module variables or depend on render order. https://react.dev/learn/keeping-components-pure

## React calls components and Hooks

- Render `<Article />`, never call `Article()`. https://react.dev/reference/rules/react-calls-components-and-hooks
- Define components and Hooks at module level. A component made inside another, or by a factory such as `makeButton(color)`, is a new type each call: the subtree remounts and loses state (lints `static-components`, `component-hook-factories`). https://react.dev/reference/eslint-plugin-react-hooks/lints/static-components and https://react.dev/reference/eslint-plugin-react-hooks/lints/component-hook-factories
- Pick between module-level components instead: `const Component = type === 'button' ? ButtonComponent : TextComponent`. https://react.dev/reference/eslint-plugin-react-hooks/lints/static-components
- Mock the server with canned data rather than mocking components. https://react.dev/reference/rules/react-calls-components-and-hooks

## Rules of Hooks

- Call Hooks at the top level of a component or custom Hook, before any early return. Not in conditions, loops, `&&` or ternaries, event handlers, class components, callbacks of `useMemo`, `useReducer` or `useEffect`, async functions, or `try`/`catch`/`finally`. https://react.dev/reference/rules/rules-of-hooks and https://react.dev/reference/eslint-plugin-react-hooks/lints/rules-of-hooks
- `use` is the exception: it may be conditional or in a loop, but only inside a component or Hook, and never inside `try`/`catch`. https://react.dev/reference/eslint-plugin-react-hooks/lints/rules-of-hooks
- Put the condition inside the Hook: `useEffect(() => { if (isLoggedIn) ... }, [isLoggedIn])`, or `useState(userType === 'admin' ? adminPerms : userPerms)`. https://react.dev/reference/eslint-plugin-react-hooks/lints/rules-of-hooks
- Anything named `use...` is treated as a Hook. https://react.dev/reference/rules/rules-of-hooks

## eslint-plugin-react-hooks

- It carries `rules-of-hooks`, `exhaustive-deps` and the React Compiler diagnostics. A reported component is skipped by the compiler, the rest still compiles, so violations can be fixed gradually. https://react.dev/reference/eslint-plugin-react-hooks
- Flat config: `reactHooks.configs.flat.recommended`. Legacy config: `"extends": ["plugin:react-hooks/recommended"]`. https://react.dev/blog/2025/10/07/react-compiler-1
- `recommended` holds 17 rules: `exhaustive-deps`, `rules-of-hooks`, `component-hook-factories`, `config`, `error-boundaries`, `gating`, `globals`, `immutability`, `incompatible-library`, `preserve-manual-memoization`, `purity`, `refs`, `set-state-in-effect`, `set-state-in-render`, `static-components`, `unsupported-syntax`, `use-memo`. https://react.dev/reference/eslint-plugin-react-hooks

## Lint rules in brief

| Rule | Flags | Fix |
| ---- | ----- | --- |
| `error-boundaries` | `try`/`catch` around child rendering or `use(promise)` | `<ErrorBoundary>` around `<Suspense>` |
| `immutability` | `items.push(4); setItems(items)`, `setItems(items.sort())` | New arrays and objects at each changed level |
| `refs` | `ref.current` read or written in render | Handlers, Effects, lazy init only |
| `set-state-in-effect` | Synchronous `setState` in an Effect | Derive in render |
| `set-state-in-render` | Unconditional `setState` in render | Guard it or set in handlers |
| `use-memo` | `useMemo` without a return value | Handler or Effect |
| `unsupported-syntax` | `eval`, `with` in components | `props[name]`, a safe parser such as `mathjs` |
| `config`, `gating` | Invalid compiler options | Valid option names and values |

https://react.dev/reference/eslint-plugin-react-hooks/lints/error-boundaries, https://react.dev/reference/eslint-plugin-react-hooks/lints/immutability, https://react.dev/reference/eslint-plugin-react-hooks/lints/refs, https://react.dev/reference/eslint-plugin-react-hooks/lints/set-state-in-effect, https://react.dev/reference/eslint-plugin-react-hooks/lints/set-state-in-render, https://react.dev/reference/eslint-plugin-react-hooks/lints/use-memo, https://react.dev/reference/eslint-plugin-react-hooks/lints/unsupported-syntax, https://react.dev/reference/eslint-plugin-react-hooks/lints/config, https://react.dev/reference/eslint-plugin-react-hooks/lints/gating

- `gating` must be an object with both `importSpecifierName` and `source`. A string is invalid. https://react.dev/reference/eslint-plugin-react-hooks/lints/gating

## Strict Mode

- Development only. It double-renders components, runs one extra Effect cycle and one extra ref callback cycle, and warns on `UNSAFE_` lifecycles. https://react.dev/reference/react/StrictMode
- Double-called: component bodies, `useState` initializers, updaters, `useMemo` and `useReducer` callbacks, class `constructor`, `render`, `shouldComponentUpdate`. Never event handlers. https://react.dev/reference/react/StrictMode
- On the second development render, `useMemo` and `useCallback` reuse the first result. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Wrap the whole app: `root.render(<StrictMode><App /></StrictMode>)`. There is no opt-out inside a subtree, move `<StrictMode>` lower instead. https://react.dev/reference/react/StrictMode
- Below the root, the extra Effect run on initial mount is skipped. https://react.dev/reference/react/StrictMode
- With DevTools, logs of the second render are dimmed. https://react.dev/reference/react/StrictMode
- A module-level counter shows "Guest #2, #4, #6" under Strict Mode: the counter is the bug. https://react.dev/learn/keeping-components-pure
