# Configuring rules, inline config comments, disable directives

## Contents

- Severities and options
- Inline config comments
- Disable directives
- Linter options
- Fixes and suggestions
- Formatting rules

## Severities and options

- `"off"` or `0`, `"warn"` or `1` (does not change the exit code), `"error"` or `2` (exit code 1). https://eslint.org/docs/latest/use/configure/rules
- Use `"error"` for what CI, pre-commit or merge must block. Use `"warn"` while introducing a rule, or for rules prone to false positives. https://eslint.org/docs/latest/use/configure/rules
- Options: `[severity, ...options]`. Plugin rules take the namespace prefix: `"example/rule1"`. https://eslint.org/docs/latest/use/configure/rules
- A later object that sets only a severity keeps the earlier options: `["error", "never"]` then `"warn"` gives `["warn", "never"]`. https://eslint.org/docs/latest/use/configure/rules
- Turn a rule off for a file group with a later object: `{ files: ["*-test.js", "*.spec.js"], rules: { "no-unused-expressions": "off" } }`. https://eslint.org/docs/latest/use/configure/rules
- ESLint lints nothing useful until you extend a config or enable rules. https://eslint.org/docs/latest/use/getting-started

## Inline config comments

- Inline config comments apply after the config file and have the highest priority. https://eslint.org/docs/latest/use/configure/rules
- A comment cannot load a plugin. A plugin rule in a comment works only if the config already loads the plugin. https://eslint.org/docs/latest/use/configure/rules
- A comment with a severity only keeps the config options: config `curly: ["error", "multi"]` plus `/* eslint curly: "warn" */` gives `["warn", "multi"]`. Give an option to replace them: `/* eslint curly: ["warn", "all"] */`. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- Several `/* eslint */` comments for one rule in a file: the first wins, the others are reported as errors. https://eslint.org/docs/latest/use/migrate-to-9.0.0
- `/* eslint no-alert: "off" */` keeps a rule off whatever enable comments follow. https://eslint.org/docs/latest/use/configure/rules

```js
/* eslint eqeqeq: "off", curly: "error" */
/* eslint quotes: ["error", "double"], curly: 2 */
```

https://eslint.org/docs/latest/use/configure/rules

## Disable directives

- Forms: `/* eslint-disable */` ... `/* eslint-enable */`, `// eslint-disable-line rule`, `// eslint-disable-next-line a, b`. https://eslint.org/docs/latest/use/configure/rules
- A bare `/* eslint-enable */` re-enables every disabled rule, not only the last one. https://eslint.org/docs/latest/use/configure/rules
- Disabled code must still parse: ESLint parses the whole file. https://eslint.org/docs/latest/use/configure/rules
- Write the reason after 2 or more dashes. In a block comment, the line that holds the dashes must not start with `*`. https://eslint.org/docs/latest/use/configure/rules
- Disable rarely and with a reason, prefer config over comments, track temporary disables as follow-up work. `eslint-plugin-eslint-comments` enforces such practices. https://eslint.org/docs/latest/use/configure/rules

```js
// eslint-disable-next-line no-console -- printed once at startup
console.log("ready");
```

https://eslint.org/docs/latest/use/configure/rules

## Linter options

- `linterOptions.noInlineConfig: true` (CLI `--no-inline-config`) ignores every inline config comment and disable directive, `/* global */` included. https://eslint.org/docs/latest/use/configure/rules , https://eslint.org/docs/latest/use/command-line-interface
- `reportUnusedDisableDirectives` defaults to `"warn"`. A rule bug fixed in a patch release can make an old directive unused and fail a run with severity `"error"`. https://eslint.org/docs/latest/use/configure/rules , https://eslint.org/docs/latest/use/command-line-interface
- `reportUnusedInlineConfigs` (default `"off"`) reports inline config comments that repeat the configured severity and options. https://eslint.org/docs/latest/use/configure/rules

```js
export default defineConfig([
  { linterOptions: { noInlineConfig: false, reportUnusedDisableDirectives: "error" } },
]);
```

https://eslint.org/docs/latest/use/configure/configuration-files

## Fixes and suggestions

- Fixes (wrench icon) apply with `--fix` and do not change logic. Suggestions (bulb icon) may change logic: the CLI never applies them, editors do. https://eslint.org/docs/latest/use/core-concepts/
- "Circular fixes detected" (`ESLintCircularFixesWarning`) means 2 fixable rules undo each other across passes. Remove or reconfigure 1 of them. To find the pair, apply 1 fix in an editor and see which new problem appears. https://eslint.org/docs/latest/use/troubleshooting/circular-fixes

## Formatting rules

- Core formatting rules are deprecated and stylistic rules are frozen except for new syntax. Use a formatter (Prettier, dprint) or ESLint Stylistic. https://eslint.org/docs/latest/use/core-concepts/glossary
- Deprecated rules still run but get no bug fixes, enhancements or docs updates. Formatter output lists them in `usedDeprecatedRules`. https://eslint.org/docs/latest/use/rule-deprecation , https://eslint.org/docs/latest/use/formatters/
