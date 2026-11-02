# Core rules: variables, scope and possible errors

## Contents

- Rules TypeScript already covers
- Unused and unassigned variables
- Undeclared, shadowed, redeclared
- Conditions, loops, switch
- Async code and errors
- Regular expressions
- Other traps

## Rules TypeScript already covers

- The compiler already does the job of `no-class-assign`, `no-const-assign`, `no-dupe-args`, `no-dupe-class-members`, `no-dupe-keys`, `no-func-assign`, `no-import-assign`, `no-invalid-this`, `no-new-native-nonconstructor`, `no-new-symbol`, `no-obj-calls`, `no-redeclare`, `no-setter-return`, `no-this-before-super`, `no-undef`, `no-unreachable`, `no-unsafe-negation`. https://eslint.org/docs/latest/rules/no-dupe-keys , https://eslint.org/docs/latest/rules/no-undef
- Exceptions: `no-import-assign` still catches `Object.assign()` on an import. `no-redeclare` still catches `var`. `no-invalid-this` matters without `strict` or `noImplicitThis`. `no-unreachable` matters unless TypeScript is set to `allowUnreachableCode: false`. https://eslint.org/docs/latest/rules/no-import-assign , https://eslint.org/docs/latest/rules/no-redeclare , https://eslint.org/docs/latest/rules/no-invalid-this , https://eslint.org/docs/latest/rules/no-unreachable
- In strict code (modules), the parser itself reports `delete` of a variable, duplicate params, octal literals and escapes, and `with`. `no-dupe-args` never covers arrow functions or class methods. https://eslint.org/docs/latest/rules/no-delete-var , https://eslint.org/docs/latest/rules/no-dupe-args , https://eslint.org/docs/latest/rules/no-octal , https://eslint.org/docs/latest/rules/no-with

## Unused and unassigned variables

- `no-unused-vars` default: `{ "vars": "all", "args": "after-used", "caughtErrors": "all", "ignoreRestSiblings": false, "ignoreUsingDeclarations": false, "reportUsedIgnorePattern": false }`. An unused `catch (e)` is reported (`caughtErrors: "all"`). https://eslint.org/docs/latest/rules/no-unused-vars
- A variable only declared or only assigned is unused. Reading, calling, passing or reading in a callback counts as use. https://eslint.org/docs/latest/rules/no-unused-vars
- Ignore patterns per kind: `varsIgnorePattern`, `argsIgnorePattern` (`"^_"`), `caughtErrorsIgnorePattern`, `destructuredArrayIgnorePattern`. `reportUsedIgnorePattern: true` flags ignored names that are used. https://eslint.org/docs/latest/rules/no-unused-vars
- `/* exported name */` works only in scripts: not with `sourceType` `module` or `commonjs`, nor with `globalReturn`. https://eslint.org/docs/latest/rules/no-unused-vars
- `no-unassigned-vars` flags `let` or `var` read but never assigned. In TypeScript, use `declare` for intentional cases. https://eslint.org/docs/latest/rules/no-unassigned-vars
- `no-useless-assignment` flags dead stores, not variables never read (that is `no-unused-vars`). It does not report certain reassignments inside `try`. https://eslint.org/docs/latest/rules/no-useless-assignment

## Undeclared, shadowed, redeclared

- `no-undef` needs globals declared in config or `/* global */`. `typeof undeclared` is not reported (option `typeof`, default `false`). Assignment to read-only globals is `no-global-assign`. https://eslint.org/docs/latest/rules/no-undef
- `no-shadow` default `{ "builtinGlobals": false, "hoist": "functions", "allow": [], "ignoreOnInitialization": false }`. TypeScript adds `hoist: "types"` or `"functions-and-types"`, and `ignoreTypeValueShadow` and `ignoreFunctionTypeParameterNameValueShadow`, both `true`. https://eslint.org/docs/latest/rules/no-shadow
- `no-shadow` reports enum members named like an outer variable on purpose: inside the enum, `B = A` means `Test.A`. https://eslint.org/docs/latest/rules/no-shadow
- `no-redeclare` has `builtinGlobals: true`. In CommonJS the top scope is module scope, so a builtin name is shadowing: use `no-shadow` with `builtinGlobals`. https://eslint.org/docs/latest/rules/no-redeclare
- `no-shadow-restricted-names` covers `globalThis`, `NaN`, `Infinity`, `undefined`, `eval`, `arguments`. https://eslint.org/docs/latest/rules/no-shadow-restricted-names
- `no-use-before-define` default `{ functions: true, classes: true, variables: true, allowNamedExports: false, enums: true, typedefs: true, ignoreTypeReferences: true }`. `"nofunc"` means `functions: false`. https://eslint.org/docs/latest/rules/no-use-before-define
- `no-invalid-this` applies in strict code only. Top-level `this` is invalid in modules. Capitalized functions count as constructors (`capIsConstructor: false` disables that). https://eslint.org/docs/latest/rules/no-invalid-this
- `no-implicit-globals` targets browser scripts and skips `const`, `let`, `class` unless `lexicalBindings: true`. https://eslint.org/docs/latest/rules/no-implicit-globals

## Conditions, loops, switch

- `no-constant-condition` `checkLoops` defaults to `"allExceptWhileTrue"`, so `while (true)` passes. https://eslint.org/docs/latest/rules/no-constant-condition
- `no-constant-binary-expression` also flags comparison with a new object, array or function, which never matches. https://eslint.org/docs/latest/rules/no-constant-binary-expression
- `no-cond-assign` defaults to `"except-parens"`: `if ((x = y))` passes. https://eslint.org/docs/latest/rules/no-cond-assign
- `no-fallthrough` accepts a comment matching `/falls?\s?through/i`, never a directive. `reportUnusedFallthroughComment` flags misleading ones. https://eslint.org/docs/latest/rules/no-fallthrough
- `no-case-declarations`: wrap `case` bodies that declare `let`, `const`, `function` or `class` in a block. https://eslint.org/docs/latest/rules/no-case-declarations
- `no-unreachable-loop` does not evaluate conditions: `if (true) break;` is missed. https://eslint.org/docs/latest/rules/no-unreachable-loop
- `for-direction` does not check that the loop runs at all. https://eslint.org/docs/latest/rules/for-direction
- `no-unexpected-multiline`: ASI does not end the statement when the next line starts with `[`, `(`, `+`, `*`, `/`, `-`, `,` or `.`, so two lines become one expression. `semi` does not flag the patterns this rule reports. https://eslint.org/docs/latest/rules/no-unexpected-multiline
- `use-isnan` checks `switch` by default but not `indexOf(NaN)` (`enforceForIndexOf: true` covers `indexOf` and `lastIndexOf`, even on a non-array object with such a method). https://eslint.org/docs/latest/rules/use-isnan
- `no-loop-func` cannot see that a function is called and dropped at once (`some` callback). Disable per line there. https://eslint.org/docs/latest/rules/no-loop-func
- `array-callback-return` matches by method name, arrays or not. `checkForEach` defaults to `false`. https://eslint.org/docs/latest/rules/array-callback-return

## Async code and errors

- `no-await-in-loop`: start the promises, then `await Promise.all(promises)`. Turn it off when iterations must be serial (retries, rate limits, dependent results). https://eslint.org/docs/latest/rules/no-await-in-loop
- `no-async-promise-executor`: errors thrown in an async executor are lost. https://eslint.org/docs/latest/rules/no-async-promise-executor
- `require-atomic-updates` reports `total += await f()`: the read happens before the pause. Read into a local first. https://eslint.org/docs/latest/rules/require-atomic-updates
- `preserve-caught-error` requires `cause` when throwing in `catch`: `throw new Error("Failed", { cause: error })`. Custom classes go in `errorClassNames` (`{ "name": "APIError", "argumentPosition": 3 }`). It matches names without scope analysis. https://eslint.org/docs/latest/rules/preserve-caught-error
- `prefer-promise-reject-errors` reports only when sure, and not throws in async functions (`no-throw-literal` covers those). https://eslint.org/docs/latest/rules/prefer-promise-reject-errors
- `require-await`: disable it if you add `async` so that throws become rejections. https://eslint.org/docs/latest/rules/require-await
- `no-unsafe-finally` allows control flow in nested functions. Generators may override on purpose: disable per line. https://eslint.org/docs/latest/rules/no-unsafe-finally
- `no-unsafe-optional-chaining`: parentheses end short-circuiting, so `(obj?.foo)()` can throw. https://eslint.org/docs/latest/rules/no-unsafe-optional-chaining

## Regular expressions

- `no-control-regex` checks the evaluated string in `new RegExp()`: `"\t"` is reported, `"\\t"` passes. https://eslint.org/docs/latest/rules/no-control-regex
- `no-empty-character-class` ignores the `RegExp` constructor string. https://eslint.org/docs/latest/rules/no-empty-character-class
- `no-invalid-regexp` follows the latest ECMAScript spec whatever the parser settings. https://eslint.org/docs/latest/rules/no-invalid-regexp
- `require-unicode-regexp` (`requireFlag: "u"` or `"v"`): adding `u` to an `i` regex with `\w` changes matches (Unicode case folding). https://eslint.org/docs/latest/rules/require-unicode-regexp
- `prefer-regex-literals` catches missing backslashes: `new RegExp("^\d\.$")` equals `/^d.$/`. https://eslint.org/docs/latest/rules/prefer-regex-literals

## Other traps

- `no-prototype-builtins`: a JSON payload `{"hasOwnProperty": 1}` breaks `foo.hasOwnProperty()`. Use `Object.hasOwn()` or `Object.prototype.hasOwnProperty.call()`. https://eslint.org/docs/latest/rules/no-prototype-builtins , https://eslint.org/docs/latest/rules/guard-for-in
- `no-empty` skips blocks containing a comment. `allowEmptyCatch: true` allows empty `catch`. https://eslint.org/docs/latest/rules/no-empty
- `no-empty-pattern` catches `{ a: {} }` written for `{ a = {} }`. https://eslint.org/docs/latest/rules/no-empty-pattern
- `no-irregular-whitespace` skips strings only by default. https://eslint.org/docs/latest/rules/no-irregular-whitespace
- `no-unused-expressions` never flags calls or `new`. In TypeScript, `x as T` and `x!` are unused when `x` is. https://eslint.org/docs/latest/rules/no-unused-expressions
