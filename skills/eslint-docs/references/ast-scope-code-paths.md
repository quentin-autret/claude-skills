# Selectors, SourceCode, scope manager, code path analysis

## Contents

- Selectors
- SourceCode API
- Scopes from a rule
- Scope manager objects
- Code path analysis
- Profiling rules
- Old patterns

## Selectors

- Selectors work as visitor keys in rules and in `no-restricted-syntax`, which bans a pattern without writing a rule. https://eslint.org/docs/latest/extend/selectors
- Supported: node type, `*`, `[attr]`, `[attr="foo"]`, `[attr=123]`, `[attr=/re/]`, `!=`, `>`, `<`, `>=`, `<=`, nested `[a.b="x"]`, field `FunctionDeclaration > Identifier.id`, `:first-child`, `:last-child`, `:nth-child(2)`, `:nth-last-child(1)` (no `an+b`), descendant (space), `>`, `~`, `+`, `:not()`, `:matches()` / `:is()`, and the classes `:statement`, `:expression`, `:declaration`, `:function`, `:pattern`. https://eslint.org/docs/latest/extend/selectors
- Several selectors matching one node run by increasing specificity (attribute, class and pseudo-class count, `:not()` excluded, then node type count), then alphabetically. https://eslint.org/docs/latest/extend/selectors
- In a regex attribute, escape `/` as `\/`. In JSON, escape the backslash too: `\\/`. https://eslint.org/docs/latest/extend/selectors

```js
return {
  "IfStatement > BlockStatement"(node) {},
  "FunctionDeclaration[params.length>3]"(node) {},
};
```

```json
"no-restricted-syntax": ["error", "ImportDeclaration[source.value=/^some\\/path$/]"]
```

https://eslint.org/docs/latest/extend/selectors

## SourceCode API

- Get it from `context.sourceCode`. https://eslint.org/docs/latest/extend/custom-rules
- `getText(node?, beforeCount?, afterCount?)` reads what the AST lacks: commas, semicolons, parentheses. https://eslint.org/docs/latest/extend/custom-rules
- Tokens: `getFirstToken(s)`, `getLastToken(s)`, `getTokenBefore`, `getTokenAfter`, `getTokens`, `getTokensBetween`, `getTokenByRangeStart`. Options: `{ skip, includeComments, filter }`, `{ count, includeComments, filter }`. https://eslint.org/docs/latest/extend/custom-rules
- Comments: `getAllComments()`, `getCommentsBefore`, `getCommentsAfter`, `getCommentsInside`, `commentsExistBetween`. A shebang is a `"Shebang"` comment token. https://eslint.org/docs/latest/extend/custom-rules
- Others: `isSpaceBetween`, `isGlobalReference`, `getNodeByRangeIndex`, `getLocFromIndex`, `getIndexFromLoc`, `getAncestors(node)` (the node excluded), `getDeclaredVariables(node)`, `getScope(node)`, `markVariableAsUsed(name, node?)`. https://eslint.org/docs/latest/extend/custom-rules
- Properties: `text` (BOM stripped, see `hasBOM`), `ast`, `lines`, `scopeManager`, `visitorKeys`, `parserServices` (empty with the default parser). https://eslint.org/docs/latest/extend/custom-rules
- `markVariableAsUsed` tells `no-unused-vars` about a use it cannot see. Without a node it uses the top-level scope (module scope in ESM, first function scope in CommonJS). https://eslint.org/docs/latest/extend/custom-rules

## Scopes from a rule

- `sourceCode.getScope(node)` returns the innermost scope of a node with several (`Program` in a module gives the `module` scope). https://eslint.org/docs/latest/extend/custom-rules
- Block, switch and `for` scopes exist only when the parser supports block scope (default parser: `ecmaVersion >= 6`), and a `for` scope only when the loop declares a block-scoped iteration variable (`for (let i = 0;;)`). Class field initializers get a `class-field-initializer` scope. https://eslint.org/docs/latest/extend/custom-rules
- Implicit globals (assignment to undeclared names) are not in the global scope's `variables` or `set`, and their `references` is empty. Find them in the global scope's `through`. https://eslint.org/docs/latest/extend/scope-manager-interface
- Never call `scopeManager.addGlobals()` from a rule: it is reserved for ESLint. https://eslint.org/docs/latest/extend/scope-manager-interface

## Scope manager objects

- `ScopeManager`: `scopes`, `globalScope`, `acquire(node, inner = false)` (outermost scope unless `inner`, never a `function-expression-name` scope, `null` if none), `getDeclaredVariables(node)`. https://eslint.org/docs/latest/extend/scope-manager-interface
- Scope types: `block`, `catch`, `class`, `class-field-initializer`, `class-static-block`, `for`, `function`, `function-expression-name`, `global`, `module`, `switch`, `with`. `variableScope` is the nearest function-like or top-level scope. https://eslint.org/docs/latest/extend/scope-manager-interface
- `Scope`: `type`, `isStrict`, `upper`, `childScopes`, `variableScope`, `block`, `variables`, `set`, `references`, `through` (unresolved here), `functionExpressionScope`, `implicit` (global only). https://eslint.org/docs/latest/extend/scope-manager-interface
- `Variable`: `name`, `scope`, `identifiers`, `references`, `defs`. `Reference`: `identifier`, `from`, `resolved` (`null` when undeclared), `writeExpr`, `init`, `isWrite()`, `isRead()`, `isWriteOnly()`, `isReadOnly()`, `isReadWrite()`. https://eslint.org/docs/latest/extend/scope-manager-interface
- `Definition.type`: `CatchClause`, `ClassName`, `FunctionName`, `ImplicitGlobalVariable`, `ImportBinding`, `Parameter`, `Variable`. https://eslint.org/docs/latest/extend/scope-manager-interface
- Global variables also carry `writeable`, `eslintExplicitGlobal`, `eslintExplicitGlobalComments`, `eslintImplicitGlobalSetting`. https://eslint.org/docs/latest/extend/custom-rules

## Code path analysis

- Events, as visitor keys: `onCodePathStart(codePath, node)`, `onCodePathEnd`, `onCodePathSegmentStart(segment, node)`, `onCodePathSegmentEnd`, `onUnreachableCodePathSegmentStart`, `onUnreachableCodePathSegmentEnd`, `onCodePathSegmentLoop(from, to, node)`. https://eslint.org/docs/latest/extend/code-path-analysis
- A loop back edge fires `onCodePathSegmentLoop`, not `onCodePathSegmentStart`. Reachable and unreachable segments have separate events: track both. https://eslint.org/docs/latest/extend/code-path-analysis
- `CodePath`: `id`, `origin` (`"program"`, `"function"`, `"class-field-initializer"`, `"class-static-block"`), `initialSegment`, `finalSegments`, `returnedSegments`, `thrownSegments`, `upper`, `childCodePaths`, `traverseSegments()`. `CodePathSegment`: `id`, `nextSegments`, `prevSegments`, `allNextSegments`, `allPrevSegments`, `reachable`. https://eslint.org/docs/latest/extend/code-path-analysis
- These objects are shared by all rules. Never mutate them: keep your data in a map keyed by `id`. https://eslint.org/docs/latest/extend/code-path-analysis
- Code paths are computed before rules run, so arrays like `childCodePaths` are complete at any event. Compare lengths with `<` or `>`, or move the logic into `onCodePathEnd`. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Current segments pattern: push a set on `onCodePathStart`, pop on `onCodePathEnd`, add on segment start, delete on segment end, then test `segment.reachable` in a node visitor. https://eslint.org/docs/latest/extend/code-path-analysis

## Profiling rules

- Profile rules with the environment variable `TIMING=1`, `TIMING=50` or `TIMING=all`. Per file and per rule timings come from the `stats` option. https://eslint.org/docs/latest/extend/custom-rules

## Old patterns

| Old | Current |
| :-- | :-- |
| `CodePath#currentSegments` | Track segments with the start and end events |
| `scopeManager.isModule()`, `isImpliedStrict()`, `isStrictModeSupported()`, `acquireAll()` | Unused by ESLint, avoid |
| `Scope#taints`, `dynamic`, `directCallToEvalScope`, `thisFound`, `resolve()`, `isStatic()`, `isArgumentsMaterialized()`, `isThisMaterialized()`, `isUsedName()` | Unused by ESLint, avoid |
| `Variable#tainted`, `stack`, `Reference#tainted`, `flag`, `partial`, `isStatic()`, `Definition#index`, `kind` | Unused by ESLint, avoid |

https://eslint.org/docs/latest/use/migrate-to-9.0.0 , https://eslint.org/docs/latest/extend/scope-manager-interface
