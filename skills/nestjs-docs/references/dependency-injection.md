# Dependency injection, modules and lifecycle

## Contents
- Providers and tokens
- Modules and dynamic modules
- Injection scopes and AsyncLocalStorage
- Circular dependencies, ModuleRef, lazy loading
- Lifecycle events
- Discovery service
- Diagnosing resolution errors

## Providers and tokens

- Resolution uses the emitted constructor parameter type, so it must be a class. An interface, a type alias, or a class imported with `import type` (erased) fails with "Nest can't resolve dependencies". For non-classes use a token and `@Inject(TOKEN)`. https://docs.nestjs.com/providers#dependency-injection
- Prefer `Symbol` tokens in libraries and large apps, exported from one shared file, to avoid collisions between equal strings. An abstract class can be both the contract and the token, so no `@Inject()` is needed. https://docs.nestjs.com/fundamentals/custom-providers#interfaces-and-abstract-classes
  ```typescript
  export const LOGGER_SERVICE = Symbol('LOGGER_SERVICE');
  { provide: LOGGER_SERVICE, useClass: PinoLoggerService }
  constructor(@Inject(LOGGER_SERVICE) private readonly logger: LoggerService) {}
  ```
- A factory's `inject` entries map to arguments by order. An entry can be optional: `{ token: 'SomeOptionalProvider', optional: true }` resolves to `undefined`. https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory
- An `async` `useFactory` delays application start until it resolves. https://docs.nestjs.com/fundamentals/async-providers
- `@Optional()` injects `undefined` when nothing is registered. Prefer constructor injection, property injection only for base classes. https://docs.nestjs.com/providers#optional-providers
- Registering the same service in several modules creates one instance per module. Export it from one module and import that module. https://docs.nestjs.com/modules#shared-modules
- Set `NEST_DEBUG` to any non-empty value for extra resolution logs. https://docs.nestjs.com/faq/common-errors#debugging-dependency-errors

## Modules and dynamic modules

- `@Global()` modules are registered once, usually in the root module. Making everything global is discouraged. https://docs.nestjs.com/modules#global-modules
- Calling `forRoot()` in a second module creates a second, separately configured instance. Configure once and re-export the module class (not `forRoot()`). https://docs.nestjs.com/modules#dynamic-modules
- Naming guideline: `register()` per calling module, `forRoot()` once for the app, `forFeature()` adapts `forRoot()` config per module. Each usually has an `*Async()` variant. https://docs.nestjs.com/fundamentals/dynamic-modules#community-guidelines
- Use `ConfigurableModuleBuilder` instead of writing `register()`/`registerAsync()` by hand: https://docs.nestjs.com/fundamentals/dynamic-modules#configurable-module-builder
  ```typescript
  export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
    new ConfigurableModuleBuilder<IdGeneratorModuleOptions>().build();

  @Module({ providers: [IdGenerator], exports: [IdGenerator] })
  export class IdGeneratorModule extends ConfigurableModuleClass {}
  ```
- In `registerAsync()`, `useFactory`, `useClass` and `useExisting` are mutually exclusive. `imports` must contain the module exporting what `inject` lists. A `useClass` class must expose `create()` unless renamed with `setFactoryMethodName()`. https://docs.nestjs.com/fundamentals/dynamic-modules#configurable-module-builder
- Options that do not belong in the options token (such as `isGlobal`) go through `setExtras()`. https://docs.nestjs.com/fundamentals/dynamic-modules#extra-options
- To extend generated methods, type them with exported `OPTIONS_TYPE` / `ASYNC_OPTIONS_TYPE` and spread `super.register(options)`. https://docs.nestjs.com/fundamentals/dynamic-modules#extending-auto-generated-methods

## Injection scopes and AsyncLocalStorage

- Keep the default singleton scope. `REQUEST` scope bubbles up: anything depending on a request-scoped provider (controllers included) becomes request-scoped. `TRANSIENT` does not bubble. https://docs.nestjs.com/fundamentals/injection-scopes#scope-hierarchy
- Anything that injects `REQUEST` is request-scoped, whatever scope it declares. https://docs.nestjs.com/fundamentals/injection-scopes#request-provider
- When a provider is request-scoped only to read a per-request value (user, tenant, locale), use an `AsyncLocalStorage` store instead: providers stay singletons and it works the same in HTTP, message handlers and queue jobs. https://docs.nestjs.com/fundamentals/injection-scopes#performance
- Nest core has no ALS abstraction. Pattern: provide `{ provide: AsyncLocalStorage, useValue: new AsyncLocalStorage() }`, run `this.als.run(store, () => next())` in middleware bound with `.forRoutes('*path')`. A middleware store covers HTTP only. https://docs.nestjs.com/recipes/async-local-storage#custom-implementation
- Third-party option: `nestjs-cls` (`ClsModule.forRoot({ middleware: { mount: true, setup } })`, `ClsService`). https://docs.nestjs.com/recipes/async-local-storage#nestjs-cls
- Passport strategies and cron controllers must stay singletons. https://docs.nestjs.com/fundamentals/injection-scopes#usage
- In GraphQL inject `CONTEXT` (from `@nestjs/graphql`) instead of `REQUEST`. https://docs.nestjs.com/fundamentals/injection-scopes#request-provider
- WebSocket gateways support request-scoped providers in v12. The scope is the socket connection, not a message. https://docs.nestjs.com/fundamentals/injection-scopes#usage
- Durable providers (`@Injectable({ scope: Scope.REQUEST, durable: true })` plus `ContextIdFactory.apply(strategy)` in `main.ts`) keep one sub-tree per tenant. Not suited to a large number of tenants. https://docs.nestjs.com/fundamentals/injection-scopes#durable-providers

## Circular dependencies, ModuleRef, lazy loading

- Barrel files (`index.ts`) cause circular imports. Never import a sibling file through its directory barrel. https://docs.nestjs.com/fundamentals/circular-dependency
- `forwardRef()` must wrap both sides, for providers (`@Inject(forwardRef(() => CommonService))`) and modules (`imports: [forwardRef(() => CatsModule)]`). Construction order is indeterminate, and cycles with request-scoped providers can yield `undefined`. https://docs.nestjs.com/fundamentals/circular-dependency#forward-reference
- "The module at index [n] ... is undefined" can come from a circular dependency between modules, or from files importing each other for constants (move the constants to their own file). https://docs.nestjs.com/faq/common-errors#circular-dependency-error
- `moduleRef.get()` looks in the current module only and cannot return scoped providers. Use `{ strict: false }` for the whole graph and `resolve()` for scoped ones. Each `resolve()` call returns a new sub-tree unless you pass the same context id. https://docs.nestjs.com/fundamentals/module-ref#retrieving-instances
- Inside a request, reuse its sub-tree: https://docs.nestjs.com/fundamentals/module-ref#getting-current-sub-tree
  ```typescript
  const contextId = ContextIdFactory.getByRequest(this.request);
  const repo = await this.moduleRef.resolve(CatsRepository, contextId);
  ```
- Lazy-loaded modules (`LazyModuleLoader.load(() => LazyModule)`) suit serverless cold starts. Lifecycle hooks do not run in them, they cannot be global, global enhancers registered there misbehave, and controllers, resolvers and gateways do not behave as expected. https://docs.nestjs.com/fundamentals/lazy-loading-modules

## Lifecycle events

- Init hooks (`onModuleInit`, `onApplicationBootstrap`) run module by module from the deepest (and global) modules to the root, each awaited. Shutdown hooks run in reverse. https://docs.nestjs.com/fundamentals/lifecycle-events#lifecycle-events
- In v12, hooks are called by component hierarchy level, which can change the order of init and shutdown hooks compared with v11. https://docs.nestjs.com/migration-guide#lifecycle-hook-ordering
- Shutdown hooks are disabled by default. They run only on `app.close()`, or on signals after `app.enableShutdownHooks()`. https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown
- Hooks are not called on request-scoped classes. https://docs.nestjs.com/fundamentals/lifecycle-events#lifecycle-events
- `app.close()` does not end the process if timers keep it alive. On Windows `SIGTERM` never works (`SIGINT` and `SIGBREAK` do). Many apps with shutdown hooks in one process can trigger the too-many-listeners warning. https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown
- Order of shutdown phases: `onModuleDestroy()`, `beforeApplicationShutdown(signal)`, connections close, `onApplicationShutdown(signal)`. https://docs.nestjs.com/fundamentals/lifecycle-events#lifecycle-events
- An error during app creation exits the process with code 1. Pass `abortOnError: false` to have it thrown instead. https://docs.nestjs.com/first-steps#setup

## Discovery service

- `DiscoveryService.createDecorator()` creates a metadata decorator and indexes decorated classes: `discoveryService.getProviders({ metadataKey: FeatureFlag.KEY })`. Import `DiscoveryModule`. https://docs.nestjs.com/fundamentals/discovery-service#extracting-metadata

## Diagnosing resolution errors

- A provider listed in `imports` instead of `providers` throws at startup. Self-injection is not allowed. https://docs.nestjs.com/faq/common-errors#cannot-resolve-dependency-error
- An error naming `ModuleRef` in a monorepo usually means two copies of `@nestjs/core`. https://docs.nestjs.com/faq/common-errors#cannot-resolve-dependency-error
- `@Optional()` is not inherited by a subclass that redeclares its constructor in v12: redeclare it, or the subclass throws `UnknownDependenciesException`. https://docs.nestjs.com/migration-guide#optional-is-no-longer-inherited-by-a-redeclared-constructor
