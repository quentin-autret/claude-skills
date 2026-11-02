# Request pipeline: controllers, middleware, pipes, guards, interceptors, filters

## Contents
- Order of execution
- Controllers and routing
- Middleware
- Pipes
- Guards
- Interceptors
- Exception filters
- Custom decorators
- Global enhancers and APP_* tokens

## Order of execution

- Middleware, guards, interceptors (before), pipes, handler, interceptors (after), exception filters. https://docs.nestjs.com/faq/request-lifecycle
- Guards and interceptors resolve global, then controller, then route. The response side of interceptors unwinds route, controller, global. https://docs.nestjs.com/faq/request-lifecycle#interceptors
- Pipes run global, controller, route, then parameter. Across parameters, the last parameter runs first (`query`, then `params`, then `body`). https://docs.nestjs.com/faq/request-lifecycle#pipes
- Filters are the only component resolved route first, then controller, then global. The first filter that catches stops the others. https://docs.nestjs.com/faq/request-lifecycle#filters

## Controllers and routing

- Injecting `@Res()` or `@Next()` switches the route to library-specific mode: interceptors, `@HttpCode()` and `@Header()` stop working. Use `@Res({ passthrough: true })` to set headers or cookies and still return a value. https://docs.nestjs.com/controllers#library-specific-approach
  ```typescript
  @Get()
  findAll(@Res({ passthrough: true }) res: Response) {
    res.status(HttpStatus.OK);
    return [];
  }
  ```
- Routes register in declaration order on Express: `@Get(':id')` declared before `@Get('me')` silently shadows it, and `ParseIntPipe` does not help because routing picks the handler before pipes run. Declare static routes before parameterized ones. https://docs.nestjs.com/controllers#route-conflicts-and-resolution-order
- Opt-in diagnostics: `NestFactory.create(AppModule, { routeConflictPolicy: { duplicate: 'error', shadow: 'warn' }, routeResolutionStrategy: 'specificity' })`. Defaults are `'off'` and `'declaration'`. With `'error'`, conflicts are thrown together as `RouteConflictException` at `app.init()`. On Fastify, `shadow` and `'specificity'` have no effect (it already ranks by specificity). https://docs.nestjs.com/controllers#route-conflicts-and-resolution-order
- Wildcards: `@Get('abcd/*')` works on Express and Fastify through Nest's compatibility layer. Mid-route wildcards need a named wildcard on Express (`ab{*splat}cd`) and are unsupported on Fastify. https://docs.nestjs.com/controllers#route-wildcards
- `@QueryMethod()` maps the HTTP `QUERY` method (named to avoid a clash with `@Query()`). https://docs.nestjs.com/controllers#resources
- Query values are strings: a `number` annotation does not convert them. Nested or array query strings need a parser: `app.set('query parser', 'extended')` on Express, `querystringParser` on Fastify (`routerOptions`) and the Node adapter. https://docs.nestjs.com/controllers#query-parameters
- DTOs must be classes, not interfaces: pipes need the runtime metatype. https://docs.nestjs.com/controllers#request-payloads
- Sub-domain routing (`@Controller({ host })`, `@HostParam()`) needs the Express or Node adapter. https://docs.nestjs.com/controllers#sub-domain-routing
- `@Redirect(url, status)` defaults to 302. Returning `{ url, statusCode }` overrides it. https://docs.nestjs.com/controllers#redirection
- Controllers not listed in a module's `controllers` are never instantiated. https://docs.nestjs.com/controllers#getting-up-and-running

## Middleware

- Middleware is bound in `configure(consumer: MiddlewareConsumer)` of a module implementing `NestModule`, not in `@Module()`. `configure()` may be `async`. https://docs.nestjs.com/middleware#applying-middleware
- A middleware that does not end the response must call `next()`, or the request hangs. https://docs.nestjs.com/middleware
- Named wildcards: `'abcd/*splat'` does not match `abcd/` itself, use `'abcd/{*splat}'`. Paths in `exclude()` are written without the global prefix. `health/*splat` does not match `/health`, list `health` too. https://docs.nestjs.com/middleware#excluding-routes
- `app.use()` middleware cannot use DI. Bind class middleware with `.forRoutes('*')` in a module instead. https://docs.nestjs.com/middleware#global-middleware
- Only global filters (`useGlobalFilters()` or `APP_FILTER`) catch exceptions thrown in middleware. `@UseFilters()` on middleware is not supported. Async middleware must be declared `async` or return a Promise so rejections reach the exceptions layer. https://docs.nestjs.com/middleware#error-handling
- On Express, Nest registers `json` and `urlencoded` parsers. To configure them through the consumer, pass `bodyParser: false` to `NestFactory.create()`. https://docs.nestjs.com/middleware#applying-middleware

## Pipes

- `Parse*` pipes throw on `null` or `undefined` unless `optional: true`. Put `DefaultValuePipe` first: `@Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number`. https://docs.nestjs.com/pipes#providing-defaults
- `ParseUUIDPipe` accepts every UUID version by default. `version` also takes numbers and `'all'` since v12.1.2. https://docs.nestjs.com/pipes#binding-pipes
- `ArgumentMetadata` has `type`, `metatype`, `data` and `schema` (a Standard Schema). `metatype` is `Object` for interface-typed parameters. https://docs.nestjs.com/pipes#custom-pipes
- For schema validation in production use the built-in `StandardSchemaValidationPipe`, the chapter's Zod pipe is a teaching version. https://docs.nestjs.com/pipes#schema-based-validation
- In a hybrid app, `useGlobalPipes()` does not cover connected microservices by default. https://docs.nestjs.com/pipes#global-scoped-pipes

## Guards

- Guards run after all middleware and before interceptors and pipes. Returning `false` throws `ForbiddenException` (403). Throw your own exception for another response. https://docs.nestjs.com/guards
- Several `APP_GUARD` providers may exist. Each runs in registration order. `APP_*` tokens cannot be retrieved with `app.get()`. https://docs.nestjs.com/guards#binding-guards
- `Reflector.createDecorator<string[]>()` creates strongly typed metadata decorators. Read handler and class with `getAllAndOverride(Roles, [context.getHandler(), context.getClass()])` (first defined value in the order passed, so the handler here) or `getAllAndMerge` (merges, concatenating arrays). https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata

## Interceptors

- `tap()` is not called when the handler throws. Pass an observer with `error` or use `finalize()`. https://docs.nestjs.com/interceptors#aspect-interception
- Response mapping does not work with `@Res()` unless `passthrough` is on. https://docs.nestjs.com/interceptors#response-mapping
- A `timeout()` interceptor unsubscribes but does not abort work already started (a database query keeps running). https://docs.nestjs.com/interceptors#more-operators

## Exception filters

- An `Error` passed as the `response` of `HttpException` is not serialized. Pass it as `cause`, which is never serialized. https://docs.nestjs.com/exception-filters#throwing-standard-exceptions
- The built-in filter does not log `HttpException` (which inherits from `IntrinsicException`, never logged by built-in filters), nor `WsException` or `RpcException` in their contexts. Log from a custom filter. https://docs.nestjs.com/exception-filters#exceptions-logging
- `errorCode` in the options is serialized: `throw new BadRequestException('Password is too weak', { errorCode: 'WEAK_PASSWORD' })`. https://docs.nestjs.com/exception-filters#machine-readable-error-codes
- `http-errors` errors with `expose: false` (the 5xx default) send the standard reason phrase instead of the message since v12.2. https://docs.nestjs.com/exception-filters
- With Fastify use `response.send()`, not `response.json()`. https://docs.nestjs.com/exception-filters#exception-filters
- For a platform-agnostic catch-all, use `HttpAdapterHost` and resolve `httpAdapter` inside `catch()`, because it may not be available in the constructor. https://docs.nestjs.com/exception-filters#catch-everything
- When combining a catch-everything filter with a type-specific one, declare the catch-everything filter first. https://docs.nestjs.com/exception-filters#catch-everything
- Filters extending `BaseExceptionFilter` at method or controller scope must not be created with `new`. https://docs.nestjs.com/exception-filters#inheritance
- `useGlobalFilters()` does not cover gateways or hybrid apps. https://docs.nestjs.com/exception-filters#binding-filters

## Custom decorators

- `ValidationPipe` skips custom param decorators unless `validateCustomDecorators: true`. Custom decorators also accept the `schema` option. https://docs.nestjs.com/custom-decorators#working-with-pipes
- `@ApiHideProperty()` does not work inside `applyDecorators()`. https://docs.nestjs.com/custom-decorators#decorator-composition

## Global enhancers and APP_* tokens

- `useGlobalPipes()`, `useGlobalGuards()`, `useGlobalInterceptors()` and `useGlobalFilters()` cannot inject dependencies. Register with `APP_PIPE`, `APP_GUARD`, `APP_INTERCEPTOR`, `APP_FILTER` from `@nestjs/core` instead (guards: https://docs.nestjs.com/guards#binding-guards, pipes: https://docs.nestjs.com/pipes#global-scoped-pipes, interceptors: https://docs.nestjs.com/interceptors#binding-interceptors, filters: https://docs.nestjs.com/exception-filters#binding-filters):
  ```typescript
  providers: [{ provide: APP_GUARD, useClass: RolesGuard }]
  ```
- Bind by class, not instance, so DI works and Nest can reuse instances of the same class across the module, which reduces memory usage. https://docs.nestjs.com/exception-filters#binding-filters
