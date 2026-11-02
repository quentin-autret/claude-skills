# CLI and build: workspaces, libraries, compilers, SWC, REPL, generators

## Contents
- CLI install and requirements
- nest new and nest generate
- nest build and nest start
- Workspaces (monorepo)
- Libraries
- SWC
- Hot reload
- REPL
- CRUD generator
- nest-commander
- nest upgrade
- Old patterns

## CLI install and requirements

- Run build and start through the `package.json` scripts that `nest new` generates, so the local `nest` binary runs and the whole team shares one version. A global CLI is not a project dependency. https://docs.nestjs.com/cli/scripts#package-scripts
- The CLI binary needs Node.js v20.11+. The schematics behind `nest new`, `nest generate` and `nest upgrade` need Node.js v22.22.3+, v24.15+ or v26+. https://docs.nestjs.com/cli/overview#requirements
- The CLI needs a Node.js build with ICU: `node -p process.versions.icu` printing `undefined` means no ICU. https://docs.nestjs.com/cli/overview#requirements

## nest new and nest generate

- `nest new` prompts for the module system: ESM (default) scaffolds Vitest, CommonJS scaffolds Jest. Both use oxlint. https://docs.nestjs.com/cli/overview#basic-workflow
- In CI pass the project name and `--package-manager`. The module system then defaults to ESM, and NestJS Observe is set up only with `--observe`. https://docs.nestjs.com/cli/usages#nest-new
- `--strict` is on by default (with `strictPropertyInitialization` off) and has no off flag. Set `"strict": false` in `tsconfig.json` afterwards. https://docs.nestjs.com/cli/usages#nest-new
- The generated `decorator` schematic uses `Reflector.createDecorator()`. `--type` and `--crud` apply to `resource` only, which is TypeScript only. https://docs.nestjs.com/cli/usages#nest-generate
- `generateOptions.spec` as an object does not resolve aliases: `"service": false` still emits a spec for `nest g s`. Set both keys. https://docs.nestjs.com/cli/monorepo#global-generate-options
  ```json
  { "generateOptions": { "spec": { "service": false, "s": false } } }
  ```
- Precedence: command-line flags, then project `generateOptions`, then global ones. https://docs.nestjs.com/cli/monorepo#project-specific-generate-options

## nest build and nest start

- Builder: `tsc` by default in standard mode, `rspack` in a monorepo. Override with `--builder`/`-b` or `compilerOptions.builder` (string or `{ type, options }`). https://docs.nestjs.com/cli/overview#project-structure https://docs.nestjs.com/cli/monorepo#global-compiler-options
- `nest start --env-file .env` loads variables into `process.env`, is repeatable, and skips a missing file with a notice. Arguments after `--` reach `process.argv`. https://docs.nestjs.com/cli/usages#nest-start
- `manualRestart: true` enables typing `rs` to restart in watch mode (default `false`). https://docs.nestjs.com/cli/monorepo#global-compiler-options
- `assets` are not copied on incremental `--watch` compiles without `watchAssets: true`. They must sit under `src`. https://docs.nestjs.com/cli/monorepo#global-compiler-options https://docs.nestjs.com/cli/monorepo#assets
  ```json
  "assets": [{ "include": "**/*.graphql", "exclude": "**/omitted.graphql", "watchAssets": true }]
  ```
- A top-level `compilerOptions.watchAssets` overrides any `watchAssets` inside `assets`. https://docs.nestjs.com/cli/monorepo#assets

## Workspaces (monorepo)

- Modes differ only in build and library support. Generating an app or library in a standard project converts it: `src` and `test` move to `apps/<name>/`, which fails or misbehaves on a non-canonical layout. https://docs.nestjs.com/cli/monorepo#monorepo-mode
- The top-level `root` in `nest-cli.json` is the default project, used by `nest build` and `nest start` without a name. Keep it an application. https://docs.nestjs.com/cli/monorepo#workspace-projects
- A project's `compilerOptions` are not merged with the top-level ones: a project `assets` array replaces the top-level one. Use `includeLibraryAssets` to copy a library's assets. https://docs.nestjs.com/cli/monorepo#global-compiler-options
- Rspack outputs a single bundled file. Set the builder to `tsc` or `swc` for individual files. https://docs.nestjs.com/cli/monorepo#specified-compiler

## Libraries

- Built-in libraries exist only in monorepos. `nest g library my-library` asks for a prefix (default `@app` or `defaultLibraryPrefix`), creates `libs/my-library/`, registers `"type": "library"` with `"entryFile": "index"`, and writes `paths` into the root `tsconfig.json`. https://docs.nestjs.com/cli/libraries#creating-libraries https://docs.nestjs.com/cli/libraries#using-libraries
- `paths` is understood only by TypeScript. Mirror each alias in Jest `moduleNameMapper`, or tests fail at runtime though the build passes: https://docs.nestjs.com/cli/libraries#using-libraries
  ```json
  "moduleNameMapper": { "^@app/my-library(|/.*)$": "<rootDir>/../../libs/my-library/src/$1" }
  ```

## SWC

- `npm i --save-dev @swc/cli @swc/core`, then `nest start -b swc -w` or `"builder": "swc"`. https://docs.nestjs.com/recipes/swc#installation
- SWC does no type checking. `--type-check` (or `typeCheck: true`) runs `tsc --noEmit` alongside, and is also what runs the CLI plugins and writes their metadata file. https://docs.nestjs.com/recipes/swc#type-checking https://docs.nestjs.com/recipes/swc#cli-plugins-swc
- Output follows `package.json` `"type"`: ESM with `"module"`, CommonJS otherwise. A `module` in `.swcrc` overrides it. https://docs.nestjs.com/recipes/swc#swc-configuration
- Circular imports break under SWC. Wrap relation types (`profile: Relation<Profile>` from `typeorm`, or `type WrapperType<T> = T`) and every `forwardRef` injection: https://docs.nestjs.com/recipes/swc#common-pitfalls
  ```typescript
  @Inject(forwardRef(() => ProfileService)) private readonly profileService: WrapperType<ProfileService>
  ```
- Jest with `@swc/jest` needs `jsc.transform.legacyDecorator` and `decoratorMetadata` set to `true` in `.swcrc`. https://docs.nestjs.com/recipes/swc#jest--swc
- Vitest with `unplugin-swc`: set `swc.vite({ module: { type: 'es6' } })` explicitly so `.swcrc` is not inherited. https://docs.nestjs.com/recipes/swc#configuration
- Vitest ignores TypeScript path aliases: add `resolve.alias: { src: resolve(import.meta.dirname, './src') }`. https://docs.nestjs.com/recipes/swc#path-aliases
- In ESM e2e tests use `import request from 'supertest'`, as a namespace import is not callable. https://docs.nestjs.com/recipes/swc#update-imports-in-e2e-tests

## Hot reload

- The recipe targets webpack-based CommonJS projects. Webpack does not copy assets and breaks static glob paths such as TypeORM `entities`. https://docs.nestjs.com/recipes/hot-reload
- Webpack does not await `dispose()`, so stash `app.close()` and await it in the next instance, with `forceCloseConnections`, to avoid `EADDRINUSE`: https://docs.nestjs.com/recipes/hot-reload#hot-module-replacement
  ```typescript
  if (module.hot?.data?.closePromise) { await module.hot.data.closePromise; }
  const app = await NestFactory.create(AppModule, { forceCloseConnections: !!module.hot });
  module.hot.accept();
  module.hot.dispose((data: any) => { data.closePromise = app.close(); });
  ```

## REPL

- Add `repl.ts` calling `await repl(AppModule)` (from `@nestjs/core`) and run `npm run start -- --entryFile repl`. https://docs.nestjs.com/recipes/repl#usage
- `resolve(token, contextId)` reaches transient and request-scoped providers. https://docs.nestjs.com/recipes/repl#native-functions
- In `--watch` mode history is lost on each reload: call `replServer.setupHistory(".nestjs_repl_history", cb)` on the returned server. https://docs.nestjs.com/recipes/repl#watch-mode

## CRUD generator

- `nest g resource users` generates module, controller, resolver or gateway, service, entity, DTOs and specs. Services are ORM-agnostic placeholders. https://docs.nestjs.com/recipes/crud-generator#generating-a-new-resource

## nest-commander

- Third-party package. Commands extend `CommandRunner` with `@Command()`, are providers, and start with `await CommandFactory.run(AppModule)`. https://docs.nestjs.com/recipes/nest-commander#a-command-file
- The Nest logger is off under `CommandFactory`: pass `['warn', 'error']` as second argument. It closes the app itself when the command ends. https://docs.nestjs.com/recipes/nest-commander#running-the-command
- Test with `CommandTestFactory` from `nest-commander-testing`. https://docs.nestjs.com/recipes/nest-commander#testing

## nest upgrade

- Run `npm i -g @nestjs/cli@latest` first: `nest upgrade` (alias `nest update`) bumps only the local CLI and ships with the CLI. Use `--dry-run` to preview. https://docs.nestjs.com/cli/usages#nest-upgrade
- From a v11 project it refuses Node.js without `require(esm)`, bumps `@nestjs/*`, TypeScript to v6 and Jest to v30, migrates `webpack` options to rspack and GraphQL `playground` to `graphiql`, then prints notes on what it cannot migrate. https://docs.nestjs.com/cli/usages#nest-upgrade
- Jest can `require()` the ESM-only v12 packages only on Node.js v24.9+. https://docs.nestjs.com/cli/usages#nest-upgrade
- It does not migrate to ESM, Vitest or oxlint. https://docs.nestjs.com/cli/usages#nest-upgrade

## Old patterns

| Old | Current |
| --- | --- |
| `compilerOptions.webpack: true`, `--webpack` | `builder: "rspack"`, `--builder rspack` |
| `webpackConfigPath`, `--webpackPath` | `builder.options.configPath` |
| `swc-loader` with a webpack monorepo config, `PluginMetadataGenerator` by hand | Rspack builder (compiles with SWC built in) |
| `import * as request from 'supertest'` | `import request from 'supertest'` in ESM |

https://docs.nestjs.com/cli/monorepo#global-compiler-options https://docs.nestjs.com/cli/usages#nest-build https://docs.nestjs.com/recipes/swc#monorepo https://docs.nestjs.com/recipes/swc#update-imports-in-e2e-tests
