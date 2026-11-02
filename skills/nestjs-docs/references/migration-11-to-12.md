# Migration from NestJS 11 to 12

## Upgrade order and nest upgrade

- Update the global CLI first: `nest upgrade` bumps only the local `@nestjs/cli`, and the command ships with the CLI. https://docs.nestjs.com/cli/usages#nest-upgrade
  ```bash
  npm i -g @nestjs/cli@latest @nestjs/schematics@latest
  nest upgrade --dry-run
  nest upgrade
  ```
  https://docs.nestjs.com/migration-guide#upgrading-packages
- Run `nest upgrade` (alias `nest update`) from the root of the v11 project. Start with `--dry-run`. It replaces the manual `npm-check-updates` flow. All Nest packages must move to the same major at once. https://docs.nestjs.com/migration-guide#upgrading-packages
- What it does: https://docs.nestjs.com/cli/usages#nest-upgrade
  - refuses Node.js without `require(esm)` (older than v20.19, or v22.x older than v22.12);
  - bumps every recognized `@nestjs/*` package to its v12 major (`@nestjs/graphql`, `@nestjs/apollo`, `@nestjs/mercurius` go to v14) and reports unrecognized `@nestjs/*` packages;
  - bumps TypeScript to v6 and raises `engines.node` to `>=20.19.0`;
  - moves `nest-cli.json` off `webpack`/`webpackConfigPath` onto `--builder rspack` and updates matching `package.json` scripts;
  - renames GraphQL `playground` to `graphiql`, switches subscriptions to `graphql-ws`, swaps the NATS package and rewrites `nats` imports;
  - moves `@nestjs/config` library settings under `validationOptions.libraryOptions` and bumps Joi to v18;
  - bumps Jest to v30 (with `@types/jest`, `ts-jest`) where present;
  - checks `tsconfig.json` and `tsconfig.build.json` for settings TypeScript 6 or ESM-only packages reject (legacy `moduleResolution`, missing `rootDir`);
  - prints notes on changes it cannot migrate (lifecycle hook ordering, pipe signatures, structured logging params), installs dependencies unless `--skip-install`, then prints a report.
- Other options: `--observe`/`--no-observe` (the prompt to wire `@nestjs/observe` defaults to no), `--tag`/`-t` for an npm dist-tag such as `next`, `--collection`. https://docs.nestjs.com/cli/usages#nest-upgrade
- It deliberately does not move the project to ESM, Vitest or oxlint. Those are defaults of new v12 projects only. https://docs.nestjs.com/cli/usages#nest-upgrade
- Manual local upgrade: `npm i -D @nestjs/cli@latest @nestjs/schematics@latest typescript@^6`. `@nestjs/schematics` v12 has TypeScript 6 as peer dependency. On a peer conflict from a lockfile pinning old versions, run `npm update ts-jest typescript-eslint`. https://docs.nestjs.com/migration-guide#upgrading-packages

## Node.js and ESM-only packages

- Running a Nest 12 app needs Node.js v20.19+ (v22.12+ on 22.x). 21.x is unsupported. https://docs.nestjs.com/migration-guide#nodejs-requirements
- `nest new`, `nest generate` and `nest upgrade` need v22.22.3+, v24.15+ or v26+ (23.x, 25.x and earlier 22.x/24.x excluded). https://docs.nestjs.com/migration-guide#nodejs-requirements
- All v12 core packages are ESM-only. CommonJS apps keep working through `require(esm)`, so moving your app to ESM is optional. Review custom bootstrap scripts, build tooling, test runners and bundler configs. https://docs.nestjs.com/migration-guide#esm-packages
- AWS Lambda disables `require(esm)` on its Node 20/22/24 runtimes: set `NODE_OPTIONS=--experimental-require-module`, appended to any existing value. https://docs.nestjs.com/migration-guide#nodejs-requirements

## Testing stack

- Jest loads the ESM-only v12 packages only on Node.js v24.9+. Older versions fail with `ERR_REQUIRE_ASYNC_MODULE`. https://docs.nestjs.com/migration-guide#testing-stack
- `@nestjs/testing` stays test-runner agnostic. When moving to Vitest, update test scripts, replace runner globals and check `supertest` default imports. https://docs.nestjs.com/migration-guide#testing-stack

## Behavior changes to review

- Lifecycle hooks are called by component hierarchy level. The order of `onModuleInit`, `onApplicationBootstrap` and shutdown hooks may change: review logic and tests that depend on it. https://docs.nestjs.com/migration-guide#lifecycle-hook-ordering
- `@Optional()` is not inherited by a redeclared constructor. A subclass with its own constructor now throws `UnknownDependenciesException` where v11 resolved `undefined`. Redeclare the marker. From v12.1 a subclass without its own constructor keeps the parent markers (12.0.x dropped them too). https://docs.nestjs.com/migration-guide#optional-is-no-longer-inherited-by-a-redeclared-constructor
  ```typescript
  constructor(@Optional() options?: Options) { super(options); }
  ```
  https://docs.nestjs.com/migration-guide#optional-is-no-longer-inherited-by-a-redeclared-constructor
- `ConsoleLogger` structured params are on by default: plain objects after the message attach to the same entry (`logger.log('User created', { userId: 1 })`). In JSON mode they nest under `params`, or spread with `flattenParams`. `structuredParams: false` restores the v11 output. https://docs.nestjs.com/migration-guide#structured-logging-params
- Pipe `transform` signatures are refined and `ArgumentMetadata` is generic. The HTTP adapter error mapping is reworked. The Express adapter drains in-flight requests on shutdown. https://docs.nestjs.com/migration-guide#other-notable-release-changes

## Package-specific breaking changes

- Terminus: the legacy `HealthIndicator`/`HealthCheckError` API is removed. Inject `HealthIndicatorService` and return `indicator.up()` or `indicator.down({...})`. Throwing no longer reports unhealthy, except inside `.attempt()`. https://docs.nestjs.com/migration-guide#terminus-module
  ```typescript
  const indicator = this.healthIndicatorService.check(key);
  return indicator.attempt(() => this.dogService.ping()).withTimeout(1000);
  ```
  https://docs.nestjs.com/migration-guide#terminus-module
- Terminus: the `timeout` option of database, microservice and gRPC indicators is deprecated. Use `.withTimeout(ms)`. https://docs.nestjs.com/migration-guide#terminus-module
- `@nestjs/config`: `validationSchema` takes any Standard Schema (Zod recommended for new projects). Joi needs v18+. Library options move from `validationOptions` to `validationOptions.libraryOptions`. Joi defaults stay `allowUnknown: true`, `abortEarly: false`, merged with yours. https://docs.nestjs.com/migration-guide#config-module
  ```typescript
  ConfigModule.forRoot({ validationSchema: z.object({ PORT: z.coerce.number().default(3000) }) });
  ```
  https://docs.nestjs.com/migration-guide#config-module
- GraphQL: GraphiQL is the default IDE, configured by a `graphiql` options object. `subscriptions-transport-ws` support is removed: use `subscriptions: { 'graphql-ws': true }`. https://docs.nestjs.com/migration-guide#graphql-ide-configuration and https://docs.nestjs.com/migration-guide#graphql-subscriptions-transport
- NATS v3: `npm uninstall nats`, `npm install @nats-io/transport-node`, helpers from `import * as nats from '@nats-io/nats-core'`. `StringCodec` and `JSONCodec` are gone. Packets are serialized as JSON strings and custom deserializers receive the full message object, read with `msg.json()`. https://docs.nestjs.com/migration-guide#nats-v3
- CLI: Rspack is the default monorepo bundler. `--webpack`/`--webpackPath` and `webpack`/`webpackConfigPath` in `nest-cli.json` are deprecated for `--builder rspack`. The `angular` schematic is removed. https://docs.nestjs.com/migration-guide#webpack-deprecation-in-cli-workflows

## Moving your own code to ESM

- Add `"type": "module"` to `package.json`. Projects from `@nestjs/schematics` 11.0.6+ already use `"module": "nodenext"` and `"moduleResolution": "nodenext"`. Older ones change both, plus `resolvePackageJsonExports: true` and `target: ES2023`. https://docs.nestjs.com/migration-guide#switching-your-project-to-esm
- `"type": "module"` changes the emission of every `.ts` file and surfaces all missing-extension errors at once: fix them in one pass. https://docs.nestjs.com/migration-guide#switching-your-project-to-esm
- Relative imports take `.js` even for `.ts` sources (`'./app.module.js'`). Use `import.meta.dirname` for `__dirname`, and `createRequire(import.meta.url)` from `node:module` for `require()`. https://docs.nestjs.com/migration-guide#moving-your-own-code-to-esm
- Docs samples use ESM. In a CommonJS project drop `.js`, keep `__dirname` and call `bootstrap()` without `await`. https://docs.nestjs.com/migration-guide#moving-your-own-code-to-esm

## Old patterns

- Manual `npm-check-updates` bump: use `nest upgrade`. https://docs.nestjs.com/migration-guide#upgrading-packages
- `extends HealthIndicator` and `throw new HealthCheckError(...)`: use `HealthIndicatorService.check(key)`. https://docs.nestjs.com/migration-guide#terminus-module
- Indicator `timeout` option: use `.withTimeout(ms)`. https://docs.nestjs.com/migration-guide#terminus-module
- GraphQL `playground` or `graphiql: true`: use a `graphiql` options object. https://docs.nestjs.com/migration-guide#graphql-ide-configuration
- `subscriptions-transport-ws`: use `graphql-ws`. https://docs.nestjs.com/migration-guide#graphql-subscriptions-transport
- `nats` package with `StringCodec`/`JSONCodec`: use `@nats-io/transport-node` and `msg.json()`. https://docs.nestjs.com/migration-guide#nats-v3
- Joi options directly in `validationOptions`: put them in `validationOptions.libraryOptions`. https://docs.nestjs.com/migration-guide#config-module
- Webpack builder in the CLI: use `--builder rspack`. https://docs.nestjs.com/migration-guide#webpack-deprecation-in-cli-workflows
