# HTTP platform: adapters, versioning, cookies and HTTP features

## Contents
- Choosing an adapter
- Node adapter differences
- Fastify specifics
- Versioning and global prefix
- Cookies
- Sessions
- Raw body and body parsers
- SSE, compression, MVC, static files, router module
- Shutdown and multiple servers

## Choosing an adapter

- From v12.3 the docs recommend `NodeAdapter` (`@nestjs/platform-node`) for new apps. Express is still used when no adapter is passed. In v13 `NodeAdapter` becomes the default. https://docs.nestjs.com/http/performance#choosing-an-adapter
  ```typescript
  import { NestNodeApplication, NodeAdapter } from '@nestjs/platform-node';
  const app = await NestFactory.create<NestNodeApplication>(AppModule, new NodeAdapter());
  ```
- Escape hatches: `FastifyAdapter` for Fastify plugins and route constraints, Express for Express-only APIs or packages. To stay on Express after the default changes, pass `new ExpressAdapter()` explicitly. https://docs.nestjs.com/http/performance#choosing-an-adapter https://docs.nestjs.com/http/performance#express-adapter
- With an explicit adapter, options are the third argument: `NestFactory.create(AppModule, new FastifyAdapter(), { ... })`. Options passed second are silently dropped. https://docs.nestjs.com/observability/sdk#quick-start
- Pass the platform type (`NestExpressApplication`, `NestFastifyApplication`, `NestNodeApplication`) only when you need the underlying platform API. https://docs.nestjs.com/first-steps#platform

## Node adapter differences

- Express-style helpers on `NodeRequest`/`NodeResponse`, and helmet, compression, cookie-parser, express-session, passport work. Follow Express instructions in chapters that split Express and Fastify. https://docs.nestjs.com/http/performance#express-compatibility
- Differences from Express: routes match by specificity, paths are case-sensitive (also middleware and `exclude()`), a returned string is `text/plain`, a route param over 100 characters does not match (raise with `maxParamLength`), JSON bodies with `__proto__` or `constructor.prototype` get 400. https://docs.nestjs.com/http/performance#express-compatibility
- Not available: `res.render()` (use `@Render()`), `res.download()`, `res.format()`, `req.accepts()`, `req.route`, mounting an Express app, Express settings. https://docs.nestjs.com/http/performance#express-compatibility
- Query strings are parsed flat. Nested parsing: `new NodeAdapter({ querystringParser: (s) => qs.parse(s) })`. https://docs.nestjs.com/http/performance#node-adapter-options
- `trustProxy` takes `true`, addresses or subnets (`'loopback'`, `'10.0.0.0/8'`) or a function. A number trusts no proxy. https://docs.nestjs.com/http/performance#node-adapter-options
- Upload interceptors and `MulterModule` come from `@nestjs/platform-node`. GraphQL needs `@nestjs/apollo` v14.1.1 or later, `@nestjs/mercurius` needs Fastify. https://docs.nestjs.com/http/performance#other-packages

## Fastify specifics

- Fastify listens on `localhost` only: `app.listen(3000, '0.0.0.0')` to accept other hosts. https://docs.nestjs.com/http/performance#adapter
- Fastify v5.12.1 and later (installed by `@nestjs/platform-fastify` v12) trusts no proxy when `trustProxy` is a number. Pass the proxy address or range. https://docs.nestjs.com/security/rate-limiting#proxies
- Fastify has no nested routers, so no host-based routing. Middleware receives raw `req`/`res`. https://docs.nestjs.com/controllers#sub-domain-routing https://docs.nestjs.com/http/performance#middleware
- `@fastify/cors` allows only `GET,HEAD,POST` by default, Express `cors` also `PUT,PATCH,DELETE`. Set `methods` explicitly. https://docs.nestjs.com/security/cors#default-allowed-methods

## Versioning and global prefix

- URI versioning inserts the version after the global prefix with a `v` prefix (`prefix: false` disables it). Media type `key` must include the separator (`'v='`). https://docs.nestjs.com/http/versioning#uri-versioning-type
- With versioning on and no `defaultVersion`, unversioned routes behave as `VERSION_NEUTRAL`: URI versioning serves them only at the unversioned path. https://docs.nestjs.com/http/versioning#usage
- Register neutral routes after versioned ones sharing a path. On Express with non-URI versioning, same-path routes are tried in registration order. https://docs.nestjs.com/http/versioning#version-neutral
- Header, media type and custom versioning run middleware for every version. Use a guard or interceptor for version-specific code. https://docs.nestjs.com/http/versioning#middleware-versioning
- `setGlobalPrefix('v1', { exclude: [...] })`: a bare `*` wildcard is deprecated, `health/*splat` does not exclude `/health`, `health/{*splat}` excludes both. On Express, a path neither prefixed nor excluded gets Express's default 404, not Nest's. https://docs.nestjs.com/faq/global-prefix

## Cookies

- From v12.1 cookies are built in on every adapter: `@Cookies()`, `@SignedCookies()`, `httpAdapter.setCookie()`/`clearCookie()`, and the `cookies` option. https://docs.nestjs.com/http/cookies
  ```typescript
  const app = await NestFactory.create(AppModule, { cookies: { secret: process.env.COOKIE_SECRET } });
  this.adapterHost.httpAdapter.setCookie(res, 'uid', String(user.id), {
    signed: true, httpOnly: true, secure: true, sameSite: 'lax', maxAge: 60 * 60,
  });
  ```
- `maxAge` is in seconds (Express `res.cookie()` uses milliseconds, so a carried-over value lives 1000 times longer). Only `path: '/'` has a default: `httpOnly` and `secure` are `false`. https://docs.nestjs.com/http/cookies#setting-and-clearing-cookies
- Read trusted values with `@SignedCookies()`. Signing protects integrity only, and the name is not signed. https://docs.nestjs.com/http/cookies#signing-cookies
- A `secret` that is `undefined` (unset env var) starts the app and fails at request time. Validate config at startup. https://docs.nestjs.com/http/cookies#signing-cookies
- Invalid cookie names or attributes throw `TypeError` (500). `sameSite: 'none'` and `partitioned` require `secure: true`. `__Host-` rules are not enforced. https://docs.nestjs.com/http/cookies#invalid-names-and-attributes
- Rotate secrets with an array, new one first. https://docs.nestjs.com/http/cookies#rotating-secrets

## Sessions

- express-session's default store leaks memory and does not scale past one process. Set `resave` explicitly. https://docs.nestjs.com/http/session#use-with-express-default

## Raw body and body parsers

- `NestFactory.create(AppModule, { rawBody: true })` exposes `req.rawBody` (type `RawBodyRequest<Request>`). It needs the built-in parsers: never set `bodyParser: false`. https://docs.nestjs.com/faq/raw-body
- Limits: Express `100kb`, Fastify 1 MiB. Raise with `app.useBodyParser('json', { limit: '10mb' })` on Express, `app.useBodyParser('application/json', { bodyLimit })` on Fastify. https://docs.nestjs.com/faq/raw-body

## SSE, compression, MVC, static files, router module

- `@Sse()` must return an `Observable` or a Promise that resolves to one. If the client leaves while a returned Promise is still resolving, Nest never subscribes and teardown never runs: inject `@SseSignal() signal: AbortSignal` and check `signal.aborted` after async setup. https://docs.nestjs.com/http/server-sent-events#client-disconnection
- For busy sites, compress at the reverse proxy. `@fastify/compress` Brotli quality defaults to 4. https://docs.nestjs.com/http/compression
- On Fastify, `@Render()` needs the extension (`'index.hbs'`). https://docs.nestjs.com/http/mvc#fastify
- `ServeStaticModule` answers every path with `index.html` by default (`renderPath` matches all paths), while controller routes are still handled. Adjust `renderPath` and `serveRoot`. https://docs.nestjs.com/recipes/serve-static#configuration
- `RouterModule.register([{ path, module, children }])` prefixes all controllers of a module. Use carefully, overuse makes code hard to maintain. https://docs.nestjs.com/recipes/router-module

## Shutdown and multiple servers

- Keep-alive connections can keep the application from exiting. `forceCloseConnections: true` makes it exit without waiting for them (Fastify needs v12.1). https://docs.nestjs.com/faq/keep-alive-connections
- Servers you create yourself (HTTP plus HTTPS on one Express instance) are not closed by `app.close()`: close them in `onApplicationShutdown()`. https://docs.nestjs.com/faq/multiple-servers#multiple-simultaneous-servers
- `HttpAdapterHost` is not the adapter: read `adapterHost.httpAdapter`. https://docs.nestjs.com/faq/http-adapter
