# Core rules: suggestions, defaults that surprise, layout, removed rules

## Contents

- Suggestion rules with surprising defaults
- Layout rules
- Old patterns: removed rules
- Old patterns: deprecated options

## Suggestion rules with surprising defaults

- `eqeqeq` (`"always"`) also compares `null` strictly unless `{ "null": "ignore" }`. `--fix` fixes only `typeof` comparisons and same-type literals, otherwise it offers a suggestion that may change behavior. https://eslint.org/docs/latest/rules/eqeqeq
- `curly` defaults to `"all"`. With a `multi*` option (`"multi"`, `"multi-line"`, `"multi-or-nest"`), add `"consistent"` to make an if/else chain agree. https://eslint.org/docs/latest/rules/curly
- `prefer-const`: `destructuring: "any"` by default. Set `ignoreReadBeforeAssign: true` to avoid fighting `no-use-before-define`. https://eslint.org/docs/latest/rules/prefer-const
- `prefer-destructuring` guesses by key: `obj[0]` counts as array access, `arr[i]` as object access. `--fix` handles only object destructuring in declarations. https://eslint.org/docs/latest/rules/prefer-destructuring
- `func-style` defaults to `"expression"`, so function declarations are reported. `allowArrowFunctions` works only with `"declaration"`. https://eslint.org/docs/latest/rules/func-style
- `func-names` (`"always"`) also requires names on `export default` function declarations. https://eslint.org/docs/latest/rules/func-names
- `camelcase` ignores leading and trailing underscores, ALL_CAPS and calls. For snake_case API keys, use `dot-notation` `allowPattern`. https://eslint.org/docs/latest/rules/camelcase , https://eslint.org/docs/latest/rules/dot-notation
- `class-methods-use-this` checks class fields by default (`enforceForClassFields: true`). A method that ignores `this` can sometimes become an ordinary function or a static method, which changes the API. https://eslint.org/docs/latest/rules/class-methods-use-this
- `no-implicit-coercion` flags `!!`, `+foo` and `"" + foo` by default. Allowing `+` allows both `+foo` and `"" + foo`. https://eslint.org/docs/latest/rules/no-implicit-coercion
- `no-magic-numbers` has `ignore: []` by default. Bigints go in as strings (`"100n"`). https://eslint.org/docs/latest/rules/no-magic-numbers
- `no-param-reassign` has `props: false`: parameter property mutation passes unless `props: true`. https://eslint.org/docs/latest/rules/no-param-reassign
- `no-underscore-dangle`: prefer `#private` class members. https://eslint.org/docs/latest/rules/no-underscore-dangle
- `capitalized-comments` defaults to `"always"`. https://eslint.org/docs/latest/rules/capitalized-comments
- `strict` (`"safe"`) means `"global"` for CommonJS and `"function"` otherwise. In modules every `"use strict"` is reported. `--fix` only removes directives. https://eslint.org/docs/latest/rules/strict
- `sort-imports`: `--fix` sorts members on 1 line, never declarations. `memberSyntaxSortOrder` needs all 4 entries. https://eslint.org/docs/latest/rules/sort-imports
- `sort-keys`: `["error", "asc", { caseSensitive: true, natural: false, minKeys: 2 }]`. https://eslint.org/docs/latest/rules/sort-keys
- `padding-line-between-statements` does nothing until configured. The last matching config wins. https://eslint.org/docs/latest/rules/padding-line-between-statements
- `one-var` defaults to `"always"`: 1 declaration per scope. https://eslint.org/docs/latest/rules/one-var
- `yoda` defaults to `"never"`. `onlyEquality` already includes `exceptRange`. https://eslint.org/docs/latest/rules/yoda
- `no-var`: in TypeScript only `var` works for every global declaration form. https://eslint.org/docs/latest/rules/no-var
- Need a recent ES target: `prefer-object-has-own` (ES2022), `prefer-object-spread` and `prefer-named-capture-group` (ES2018), `prefer-exponentiation-operator` (ES2016), `no-empty-static-block` (ES2022). https://eslint.org/docs/latest/rules/prefer-object-has-own , https://eslint.org/docs/latest/rules/prefer-object-spread , https://eslint.org/docs/latest/rules/prefer-named-capture-group , https://eslint.org/docs/latest/rules/prefer-exponentiation-operator , https://eslint.org/docs/latest/rules/no-empty-static-block

## Layout rules

- Prefer a formatter or ESLint Stylistic over core layout rules. Know these defaults if you keep them. https://eslint.org/docs/latest/use/core-concepts/glossary
- `indent`: 4 spaces, `SwitchCase: 0` (`case` not indented). `ignoredNodes` takes selectors. https://eslint.org/docs/latest/rules/indent
- `quotes` `"double"`, `semi` `"always"`, `comma-dangle` `"never"`, `quote-props` `"always"`, `brace-style` `"1tbs"`, `arrow-parens` `"always"`, `padded-blocks` `"always"`, `object-curly-spacing` `"never"`, `operator-linebreak` `"after"` (`?` and `:` before). https://eslint.org/docs/latest/rules/quotes , https://eslint.org/docs/latest/rules/semi , https://eslint.org/docs/latest/rules/comma-dangle , https://eslint.org/docs/latest/rules/quote-props , https://eslint.org/docs/latest/rules/brace-style , https://eslint.org/docs/latest/rules/arrow-parens , https://eslint.org/docs/latest/rules/padded-blocks , https://eslint.org/docs/latest/rules/object-curly-spacing , https://eslint.org/docs/latest/rules/operator-linebreak
- `array-element-newline` and `function-call-argument-newline` default to `"always"`: a break between every item. https://eslint.org/docs/latest/rules/array-element-newline , https://eslint.org/docs/latest/rules/function-call-argument-newline
- `linebreak-style` (`"unix"`) fails on Windows checkouts converted to CRLF. Add `*.js text eol=lf` to `.gitattributes`. https://eslint.org/docs/latest/rules/linebreak-style
- `no-extra-parens` conflicts with `no-mixed-operators`: set `nestedBinaryExpressions: false`. https://eslint.org/docs/latest/rules/no-mixed-operators
- `no-multi-spaces` allows aligned object properties by default (`exceptions: { "Property": true }`). https://eslint.org/docs/latest/rules/no-multi-spaces

## Old patterns: removed rules

| Removed | Use |
| :-- | :-- |
| `require-jsdoc`, `valid-jsdoc` | `eslint-plugin-jsdoc` |
| `no-arrow-condition` | `no-confusing-arrow` and `no-constant-condition` |
| `no-comma-dangle`, `no-empty-class`, `no-empty-label` | `comma-dangle`, `no-empty-character-class`, `no-labels` |
| `no-extra-strict`, `global-strict` | `strict` |
| `no-reserved-keys`, `no-space-before-semi`, `no-wrap-func` | `quote-props`, `semi-spacing`, `no-extra-parens` `"functions"` |
| `space-after-keywords`, `space-before-keywords`, `space-return-throw-case` | `keyword-spacing` |
| `space-after-function-name`, `space-before-function-parentheses` | `space-before-function-paren` |
| `space-in-brackets` | `object-curly-spacing`, `array-bracket-spacing` |
| `space-unary-word-ops`, `spaced-line-comment`, `generator-star` | `space-unary-ops`, `spaced-comment`, `generator-star-spacing` |

https://eslint.org/docs/latest/rules/require-jsdoc , https://eslint.org/docs/latest/rules/no-arrow-condition , https://eslint.org/docs/latest/rules/no-wrap-func , https://eslint.org/docs/latest/rules/space-in-brackets

Rules the docs tell you to stop using or replace:

- `no-return-await`: do not use it. `return await` adds no microtask and gives a better stack trace. https://eslint.org/docs/latest/rules/no-return-await
- `indent-legacy` is a 3.x snapshot of `indent`: switch back to `indent`. https://eslint.org/docs/latest/rules/indent-legacy
- `prefer-reflect`: its intent "now seems misguided" per its page. https://eslint.org/docs/latest/rules/prefer-reflect
- Overlaps: `no-negated-in-lhs` catches the same mistake as `no-unsafe-negation`. `no-object-constructor` covers more than `no-new-object`, and `no-new-native-nonconstructor` more than `no-new-symbol`. https://eslint.org/docs/latest/rules/no-negated-in-lhs , https://eslint.org/docs/latest/rules/no-object-constructor , https://eslint.org/docs/latest/rules/no-new-native-nonconstructor

## Old patterns: deprecated options

| Old | Current |
| :-- | :-- |
| `eqeqeq` `"allow-null"` | `["error", "always", { "null": "ignore" }]` |
| `maximum` in `complexity`, `max-depth`, `max-params`, `max-statements`, `max-nested-callbacks` | `max` |
| `max-params` `countVoidThis: false` / `true` | `countThis: "except-void"` / `"always"` |
| `radix` `"always"`, `"as-needed"` | No option, a radix is always required |
| `quotes` `avoid-escape` | `avoidEscape` |
| `no-extra-boolean-cast` `enforceForLogicalOperands` | `enforceForInnerExpressions` |
| `no-mixed-requires` `true` | `{ grouping, allowCall }` |
| `line-comment-position` `applyDefaultPatterns` | `applyDefaultIgnorePatterns` |
| `object-property-newline` `allowMultiplePropertiesPerLine` | `allowAllPropertiesOnSameLine` |
| `eol-last` `"unix"`, `"windows"` | `"always"` with `linebreak-style` |

https://eslint.org/docs/latest/rules/eqeqeq , https://eslint.org/docs/latest/rules/max-params , https://eslint.org/docs/latest/rules/radix , https://eslint.org/docs/latest/rules/quotes , https://eslint.org/docs/latest/rules/no-extra-boolean-cast , https://eslint.org/docs/latest/rules/no-mixed-requires , https://eslint.org/docs/latest/rules/line-comment-position , https://eslint.org/docs/latest/rules/object-property-newline , https://eslint.org/docs/latest/rules/eol-last
