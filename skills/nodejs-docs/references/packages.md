# package.json entry points and conditions

## Contents

- exports
- Subpath patterns
- Subpath imports
- Conditional exports
- Package maps (added in v26.4.0)
- Old patterns

## exports

- Use `"exports"` for new packages. Unlisted subpaths, `package.json` included, throw `ERR_PACKAGE_PATH_NOT_EXPORTED`. It is not strong encapsulation: an absolute path still loads files. https://nodejs.org/docs/latest/api/packages.html#package-entry-points
- Adding `"exports"` to an existing package is likely breaking: list every previously supported entry and `./package.json`. https://nodejs.org/docs/latest/api/packages.html#package-entry-points
- Keep `"main"` next to `"exports"` for old tools, pointing to the same file. `"exports"` wins. https://nodejs.org/docs/latest/api/packages.html#main-entry-point-export

```json
{ "main": "./index.js", "exports": "./index.js" }
```

- `"exports": "./index.js"` equals `{ ".": "./index.js" }`. https://nodejs.org/docs/latest/api/packages.html#exports-sugar
- Targets start with `./` and contain no `..`, `.` or `node_modules` segments. https://nodejs.org/docs/latest/api/packages.html#path-rules-and-validation-for-export-targets
- Self-reference a package by name only works through its `"exports"`. https://nodejs.org/docs/latest/api/packages.html#self-referencing-a-package-using-its-name

## Subpath patterns

- `*` is plain string replacement and also matches `/`, exposing nested paths. Put `.js` on both sides to restrict to JS, and block private folders with `null`. List entries explicitly when there are few. https://nodejs.org/docs/latest/api/packages.html#subpath-patterns

```json
{ "exports": { "./features/*.js": "./src/features/*.js", "./features/private-internal/*": null } }
```

- Offer either extensioned or extensionless subpaths, not both. Extensioned ones work better with import maps. https://nodejs.org/docs/latest/api/packages.html#extensions-in-subpaths
- Trailing-slash folder mappings are removed (end-of-life), and remapping specifiers that end in `/` is deprecated: use `"./x/*"`. https://nodejs.org/docs/latest/api/deprecations.html#DEP0148 , https://nodejs.org/docs/latest/api/deprecations.html#DEP0155

## Subpath imports

- `"imports"` keys start with `#`, are private to the package, may point to external packages and support conditions. `#/` keys are allowed since v25.4.0 (backported to v24.14.0). They are the closest run-time substitute for tsconfig `paths`, which Node.js does not transform. https://nodejs.org/docs/latest/api/packages.html#subpath-imports , https://nodejs.org/docs/latest/api/typescript.html#paths-aliases

```json
{ "imports": { "#dep": { "node": "dep-node-native", "default": "./dep-polyfill.js" } } }
```

## Conditional exports

- The first matching key wins: order from most to least specific with `"default"` last. Nested objects act like nested `if`. https://nodejs.org/docs/latest/api/packages.html#conditional-exports , https://nodejs.org/docs/latest/api/packages.html#nested-conditions
- Built-in conditions: `node-addons`, `node`, `import`, `require`, `module-sync`, `default`. `module-sync` matches both `import` and `require` and needs ESM without top-level `await`. https://nodejs.org/docs/latest/api/packages.html#conditional-exports
- Always include `"default"`. Prefer `node` plus `default` over `node` plus `browser`. https://nodejs.org/docs/latest/api/packages.html#conditional-exports
- Community conditions: `types` (must be first), `browser`, `development` / `production`. Others are ignored unless enabled with `--conditions`. https://nodejs.org/docs/latest/api/packages.html#community-conditions-definitions

```bash
node --conditions=development index.js
```

- Using the `import` and `require` conditions can lead to hazards (dual package hazard, detailed in the esm page). https://nodejs.org/docs/latest/api/packages.html#conditional-exports

## Package maps (added in v26.4.0)

Experimental, needs `--experimental-package-map`. A static JSON table of `packages` with a `file:` `url` and `dependencies` decides bare-specifier resolution and blocks phantom dependencies. No cycle detection. https://nodejs.org/docs/latest/api/packages.html#package-maps

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `"main"` only | `"exports"` (keep `"main"` for old tools) | https://nodejs.org/docs/latest/api/packages.html#main-entry-point-export |
| `"exports": { "./lib/": "./lib/" }` | `"./lib/*": "./lib/*"` | https://nodejs.org/docs/latest/api/deprecations.html#DEP0148 |
| `index.js` and extension lookup for an ES module main entry | an `"exports"` or `"main"` entry with the exact extension | https://nodejs.org/docs/latest/api/deprecations.html#DEP0151 |
| tsconfig `paths` aliases (not transformed by Node.js) | `"imports"` with `#` keys | https://nodejs.org/docs/latest/api/typescript.html#paths-aliases |
