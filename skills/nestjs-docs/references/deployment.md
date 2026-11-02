# Deployment: production, standalone apps, serverless, Devtools

## Contents
- Production
- Standalone applications
- Serverless
- Devtools
- Old patterns

## Production

- NestJS 12 needs Node.js 20.19 or later, or 22.12 or later on the 22.x line. https://docs.nestjs.com/deployment#prerequisites
- `nest build` outputs to `dist` with entry `main.js`. In a Nest CLI monorepo pass the project: `npm run build my-app`. https://docs.nestjs.com/deployment#building-your-application
- If `tsconfig.json` also compiles `.ts` files in the project root, the entry becomes `dist/src/main.js`: point the start command there. https://docs.nestjs.com/deployment#building-your-application
- Run `node dist/main.js` in production. `nest start` runs `nest build` first. https://docs.nestjs.com/deployment#running-your-application
- Set `NODE_ENV=production`: Nest itself ignores it, but ecosystem libraries change behavior on it. https://docs.nestjs.com/deployment#node_envproduction
  ```bash
  NODE_ENV=production node dist/main.js
  ```
- The platform must route traffic to the port passed to `app.listen()` (`3000` in new projects unless `PORT` is set). https://docs.nestjs.com/deployment#running-your-application
- Health checks use `@nestjs/terminus`. https://docs.nestjs.com/deployment#health-checks
- Observability uses the official `@nestjs/observe` SDK through the `instrument` option: https://docs.nestjs.com/deployment#observability
  ```typescript
  const app = await NestFactory.create(AppModule, { instrument: ObserveInstrument });
  ```
- The docs Dockerfile ends with `RUN npm run build` and `CMD ["node", "dist/main"]`, and its `.dockerignore` excludes `node_modules` and `dist`. Match the `node:` base image to the Node version the project uses. https://docs.nestjs.com/deployment#dockerizing-your-application
- `nest deploy` forwards to `mau deploy` and offers to add `@nestjs/mau`. In CI (no prompts) run `npm install -g @nestjs/mau` then `mau deploy`. https://docs.nestjs.com/deployment#deploying

## Standalone applications

- `NestFactory.createApplicationContext(AppModule)` gives the IoC container with no network listener, for CRON scripts, CLIs, workers and serverless functions. https://docs.nestjs.com/standalone-applications#getting-started
- `app.get(Token)` searches every module. For strict lookup, select the module first: https://docs.nestjs.com/standalone-applications#retrieving-providers-from-static-modules
  ```typescript
  const tasks = app.select(TasksModule).get(TasksService, { strict: true });
  ```
- Middleware, guards, interceptors and pipes do not run, even global ones, when calling a controller method obtained with `app.get()`. Use `NestFactory.create()` when enhancers are needed. https://docs.nestjs.com/standalone-applications#retrieving-providers-from-static-modules
- For a dynamic module, export the registered module object and pass that same object to `select()`: https://docs.nestjs.com/standalone-applications#retrieving-providers-from-dynamic-modules
  ```typescript
  export const dynamicConfigModule = ConfigModule.register({ folder: './config' });
  const config = app.select(dynamicConfigModule).get(ConfigService, { strict: true });
  ```
- End a script with `await app.close()` so the process exits. It runs the shutdown lifecycle hooks. https://docs.nestjs.com/standalone-applications#terminating-phase

## Serverless

- Cold start dominates: compiling and bundling all dependencies cuts Nest bootstrap time substantially, and a standalone context starts faster than a full HTTP app. https://docs.nestjs.com/faq/serverless#benchmarks
- To bundle everything, the docs use `nest build --webpack` with `externals: []` and a `webpack.IgnorePlugin` for the lazy imports `@nestjs/microservices/microservices-module` and `@nestjs/websockets/socket-module` when `require.resolve` fails. https://docs.nestjs.com/faq/serverless#benchmarks
- Do not make rarely used resources async providers: each one delays every cold start. Load their module on demand with `LazyModuleLoader`: https://docs.nestjs.com/faq/serverless#runtime-optimizations
  ```typescript
  const { CacheModule } = await import('./cache.module.js');
  const moduleRef = await this.lazyModuleLoader.load(() => CacheModule);
  ```
- AWS Lambda with the Serverless framework uses `@codegenie/serverless-express` (needs `esModuleInterop: true`), handler `dist/main.handler`. Cache the bootstrapped handler across invocations: https://docs.nestjs.com/faq/serverless#example-integration
  ```typescript
  const app = await NestFactory.create(AppModule);
  await app.init();
  const expressApp = app.getHttpAdapter().getInstance();
  return serverlessExpress({ app: expressApp });
  // in the exported handler:
  server = server ?? (await bootstrap());
  ```
- The bundle must export `handler` as CommonJS. The default CLI webpack config sets `output.library.type: 'commonjs2'` as long as your `webpack.config.js` keeps `...options.output` when overriding `output`. https://docs.nestjs.com/faq/serverless#example-integration
- When minifying with `terser-webpack-plugin`, set `keep_classnames: true` or `class-validator` may misbehave. https://docs.nestjs.com/faq/serverless#example-integration
- `@nestjs/swagger` behind API Gateway can loop on redirects because `originalUrl` lacks the trailing slash. Register this middleware before `SwaggerModule.setup()`: https://docs.nestjs.com/faq/serverless#example-integration
  ```typescript
  app.use((req, res, next) => {
    if (req.originalUrl === '/swagger') req.originalUrl = '/swagger/';
    next();
  });
  ```
- Several functions sharing modules: use the CLI monorepo mode. https://docs.nestjs.com/faq/serverless#example-integration
- A standalone context (`createApplicationContext()` then `get(AppService)`) is lighter but wraps no controller method with enhancers. https://docs.nestjs.com/faq/serverless#using-standalone-application-feature

## Devtools

- Setup: `snapshot: true` in the create options, install `@nestjs/devtools-integration`, and register the module with the `http` flag off in production. The flag decides whether the introspection server starts at all, and `DevtoolsModule` must never run in production. https://docs.nestjs.com/devtools/overview
  ```typescript
  const app = await NestFactory.create(AppModule, { snapshot: true });
  // in AppModule imports:
  DevtoolsModule.register({ http: process.env.NODE_ENV !== 'production' })
  ```
- The introspection server listens on port 8000 by default: change it with the `port` option of `DevtoolsModule.register()`. https://docs.nestjs.com/devtools/overview
- Each app start prints a new sandbox session token that the Playground needs. https://docs.nestjs.com/devtools/overview#playground
- To inspect a "Cannot resolve dependency" failure (`@nestjs/core` 9.3.10+), add `abortOnError: false` next to `snapshot: true` and dump the partial graph, then drag `graph.json` into Devtools in "Preview" mode: https://docs.nestjs.com/devtools/overview#debugging-cannot-resolve-dependency-errors
  ```typescript
  bootstrap().catch((err) => {
    writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
    process.exit(1);
  });
  ```
- Save a full graph after `listen()` or `init()` with `app.get(SerializedGraph).toString()` (`SerializedGraph` and `PartialGraphHost` come from `@nestjs/core`). https://docs.nestjs.com/devtools/overview#preview-static-files
- The routes explorer shows which guards, interceptors and pipes apply to each entrypoint. The bootstrap performance page lists instantiation time per class, useful for serverless cold starts. https://docs.nestjs.com/devtools/overview#bootstrap-performance-analyzer

### CI/CD graph publishing (Enterprise plan only)

- Bootstrap in `preview: true` mode to publish: constructors and lifecycle hooks do not run, so CI needs no database. https://docs.nestjs.com/devtools/ci-cd-integration#publishing-graphs
  ```typescript
  const publish = process.env.PUBLISH_GRAPH === 'true';
  const app = await NestFactory.create(AppModule, { snapshot: true, preview: publish });
  if (publish) {
    await app.init();
    await new GraphPublisher(app).publish(publishOptions);
    await app.close();
  } else {
    await app.listen(3000);
  }
  ```
- `publishOptions` takes `apiKey`, `repository`, `owner`, `sha`, `target` (or `targetBranch` when CI gives no target SHA), `trigger` (`'pull'` or `'push'`) and `branch`. Keep `DEVTOOLS_API_KEY` in CI secrets. https://docs.nestjs.com/devtools/ci-cd-integration#other-cicd-tools
- A PR report appears only if a graph for the target branch was published before. Publish on `push` to deployment branches and on every pull request. https://docs.nestjs.com/devtools/ci-cd-integration#reports

## Old patterns

| Old | Current |
| --- | --- |
| `output.libraryTarget: 'commonjs2'` in the serverless webpack config | Default `output.library.type: 'commonjs2'` from the CLI config, kept by spreading `...options.output` (set `libraryTarget` only on older CLIs or configs that do not extend `options.output`) |

https://docs.nestjs.com/faq/serverless#example-integration
