# React Compiler

## Contents

- What it does and skips
- Installation
- Options
- Directives
- Incremental adoption
- Debugging and upgrading
- Compiling libraries

## What it does and skips

- A build-time tool that memoizes components and Hooks automatically. It works for React and React Native and relies on the Rules of React. https://react.dev/learn/react-compiler/introduction and https://react.dev/blog/2025/10/07/react-compiler-1
- It can memoize conditionally, after an early return, which `useMemo` cannot. Optional chains and array indices work as dependencies. https://react.dev/blog/2025/10/07/react-compiler-1
- It does not memoize plain functions outside components and Hooks, and memoization is not shared across components. Memoize a truly expensive shared function yourself. https://react.dev/learn/react-compiler/introduction
- Code that breaks the Rules of React is skipped, not failed: the rest still compiles. https://react.dev/learn/react-compiler/debugging
- Verify: React DevTools shows a "Memo ✨" badge, and the output imports `c as _c` from `react/compiler-runtime`. https://react.dev/learn/react-compiler/installation

## Installation

- `npm install -D babel-plugin-react-compiler@latest`. Works without options. https://react.dev/learn/react-compiler/installation
- The plugin must run first in the Babel plugin list. Options go in the second element. https://react.dev/learn/react-compiler/installation and https://react.dev/reference/react-compiler/configuration

```js
module.exports = {
  plugins: [['babel-plugin-react-compiler', { /* options */ }], /* other plugins */],
};
```

https://react.dev/reference/react-compiler/configuration

- Vite with `@vitejs/plugin-react` 6 or later: the inline `babel` option was removed, use `reactCompilerPreset` with `@rolldown/plugin-babel`. https://react.dev/learn/react-compiler/installation

```js
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';
export default defineConfig({
  plugins: [react(), babel({ presets: [reactCompilerPreset()] })],
});
```

https://react.dev/learn/react-compiler/installation

- Older `@vitejs/plugin-react`: `react({ babel: { plugins: ['babel-plugin-react-compiler'] } })`. https://react.dev/learn/react-compiler/installation
- Next.js: `reactCompiler` in the Next config. Expo SDK 54 and later enable it by default. React Router: `vite-plugin-babel`. Webpack: the community `react-compiler-webpack` loader. https://react.dev/learn/react-compiler/installation and https://react.dev/blog/2025/10/07/react-compiler-1
- Best with React 19, React 17 and 18 supported through `target`. https://react.dev/learn/react-compiler/installation
- Lint rules ship in `eslint-plugin-react-hooks`. Remove `eslint-plugin-react-compiler`. The linter works without the compiler installed. https://react.dev/blog/2025/10/07/react-compiler-1

## Options

- `compilationMode`: `'infer'` (default) compiles functions with `"use memo"` plus PascalCase components and `use` Hooks that create JSX or call Hooks. A lowercase `button()` or a `useData()` that calls no Hook is not compiled. `'annotation'` compiles only `"use memo"` functions. `'syntax'` is Flow only. `'all'` compiles every top-level function and is not recommended. https://react.dev/reference/react-compiler/compilationMode
- `target`: `'19'` (default), `'18'` or `'17'`, a string major only (not `18`, not `'18.2.0'`). 17 and 18 need `react-compiler-runtime` in `dependencies`, not dev. https://react.dev/reference/react-compiler/target
- `panicThreshold`: keep `'none'` (default) in production. `'critical_errors'` and `'all_errors'` are for debugging. A boolean is invalid. https://react.dev/reference/react-compiler/panicThreshold and https://react.dev/reference/eslint-plugin-react-hooks/lints/config
- `logger`: `{ logEvent(filename, event) }`, events `CompileSuccess`, `CompileError`, `CompileDiagnostic`, `CompileSkip`, `PipelineError`, `Timing`. The event shape may change between versions. https://react.dev/reference/react-compiler/logger
- `gating`: `{ source, importSpecifierName }`. The function is a named export returning a boolean, evaluated once at module load. Both versions ship, so the bundle grows. `source` resolves as a module or from the project root, not relative to `babel.config.js`. https://react.dev/reference/react-compiler/gating

```js
{ gating: { source: 'my-feature-flags', importSpecifierName: 'shouldUseCompiler' } }
```

https://react.dev/reference/react-compiler/gating

- The `config` lint catches typos such as `compileMode` and invalid values such as `compilationMode: 'everything'`. https://react.dev/reference/eslint-plugin-react-hooks/lints/config

## Directives

- `"use memo"` opts in, `"use no memo"` opts out and wins in every mode. https://react.dev/reference/react-compiler/compilationMode
- First statement of the function body, or of the file above imports. Single or double quotes, never backticks. A directive after other code is ignored. A function directive overrides the module one. https://react.dev/reference/react-compiler/directives and https://react.dev/reference/react-compiler/directives/use-memo
- In `infer` mode, fix the naming rather than adding `"use memo"`. https://react.dev/reference/react-compiler/directives/use-memo
- `"use no memo"` is temporary: add a comment and a tracking TODO, then remove it. Prefer project config over directives. https://react.dev/reference/react-compiler/directives/use-no-memo and https://react.dev/reference/react-compiler/directives

## Incremental adoption

1. Babel `overrides` with `test` globs per directory.
2. `compilationMode: 'annotation'` and `"use memo"` on each component and each custom Hook.
3. `gating` for runtime A/B tests.

https://react.dev/learn/react-compiler/incremental-adoption

- Fix lint violations as coverage grows, exclude problem spots with `"use no memo"`. https://react.dev/learn/react-compiler/incremental-adoption

## Debugging and upgrading

- Most problems are runtime: code that relies on memoization for correctness (Effects depending on referential equality, unstable dependency arrays). https://react.dev/learn/react-compiler/debugging
- Add `"use no memo"`. If the bug disappears, the component breaks a rule. Removing manual `useMemo`, `useCallback` or `memo` with the bug still present also points to a rule violation. Fix, remove the directive, check the badge. https://react.dev/learn/react-compiler/debugging
- Libraries with interior mutability break memoization and are skipped (lint `incompatible-library`): react-hook-form `watch` (use `useWatch({ control, name })`), TanStack Table `useReactTable`. MobX `observer` is not detected and may need `"use no memo"`. https://react.dev/reference/eslint-plugin-react-hooks/lints/incompatible-library
- `eval` and `with` in a component block compilation. https://react.dev/reference/eslint-plugin-react-hooks/lints/unsupported-syntax
- New compiler versions may memoize differently and over- or under-fire Effects. Without good end-to-end tests, pin the exact version (`--save-exact`). https://react.dev/blog/2025/10/07/react-compiler-1

## Compiling libraries

- Ship compiled code. Below React 19, add `react-compiler-runtime` to `dependencies` and set `target` to the minimum supported React major. Otherwise users see "Cannot find module 'react-compiler-runtime'". https://react.dev/reference/react-compiler/compiling-libraries
- Test the library with and without the compiler. https://react.dev/reference/react-compiler/compiling-libraries
- Library API rule: a returned function stays safe in `useMemo(() => fn(), [fn])`. Prefer immutable state plus explicit update functions. https://react.dev/reference/eslint-plugin-react-hooks/lints/incompatible-library
