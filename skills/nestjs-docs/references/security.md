# Security: headers, CSRF, CORS, rate limiting, hashing

## Contents
- Security headers
- CSRF protection
- CORS
- Rate limiting
- Encryption and password hashing
- Old patterns

## Security headers

- From v12.1, `app.useSecurityHeaders()` sets Helmet 8 defaults on Express and Fastify with no extra package. Call it right after `NestFactory.create()`: `app.use()` middleware registered before it answers without headers. Calling it after `init()`/`listen()` or twice throws. https://docs.nestjs.com/security/helmet#getting-started
  ```typescript
  const app = await NestFactory.create(AppModule);
  app.useSecurityHeaders();
  app.enableCsrfProtection();
  ```
- Headers also cover 404s, error responses, CSRF rejections and SSE. `X-Powered-By` is removed. https://docs.nestjs.com/security/helmet#getting-started
- Each option takes `false` (omit), `true` (default) or an object. `xPoweredBy: false` keeps the header. Helmet legacy names (`hsts`, `frameguard`, `noSniff`) are rejected. https://docs.nestjs.com/security/helmet#configuring-headers
- A CSP directive you set replaces the default of the same name, so include `"'self'"`. Keywords must be single-quoted. `null` removes a directive. https://docs.nestjs.com/security/helmet#content-security-policy
- CSP directive values cannot be functions (no per-request nonces, unlike Helmet). https://docs.nestjs.com/security/helmet#using-helmet-directly
- The default `upgrade-insecure-requests` breaks Swagger UI assets over plain HTTP outside local development: serve HTTPS or set `upgradeInsecureRequests: null`. https://docs.nestjs.com/security/helmet#swagger-ui-and-graphql-ides
- GraphiQL loads scripts from `unpkg.com` and an inline script, and by default only runs when `NODE_ENV` is not `production`, so relax CSP only there: https://docs.nestjs.com/security/helmet#swagger-ui-and-graphql-ides
  ```typescript
  const isProduction = process.env.NODE_ENV === 'production';
  app.useSecurityHeaders({
    contentSecurityPolicy: isProduction ? true
      : { directives: { scriptSrc: ["'self'", "'unsafe-inline'", 'https://unpkg.com'] } },
  });
  ```
- On Fastify, `reply.removeHeader()` cannot remove a header set by `useSecurityHeaders()`. Override per route with `@Header()`. https://docs.nestjs.com/security/helmet#configuring-headers

## CSRF protection

- From v12.1, `app.enableCsrfProtection()` rejects cross-origin writes using `Sec-Fetch-Site` and `Origin`, no token or cookie. It runs before middleware, body parsing and guards, on every request, including routes that do not exist. https://docs.nestjs.com/security/csrf
- Algorithm: GET, HEAD, OPTIONS always pass. `Sec-Fetch-Site: same-origin` or `none` pass, any other value (including `same-site`) is rejected. No `Sec-Fetch-Site` and no `Origin` pass (not a browser). Otherwise `Origin` must match `Host`. `X-Forwarded-Host` is ignored. https://docs.nestjs.com/security/csrf#how-it-works
- Origins allowed by CORS are not trusted for CSRF. A frontend on a sibling subdomain must be in `trustedOrigins` (exact `scheme://host[:port]`, no path, no wildcard). https://docs.nestjs.com/security/csrf#trusted-origins
  ```typescript
  app.enableCsrfProtection({
    trustedOrigins: ['https://admin.example.com'],
    exclude: [{ path: 'webhooks/stripe', method: RequestMethod.POST }],
  });
  ```
- Exclusions are written without the global prefix and match exactly (case, trailing slash). Set `version` on exclusions of URI-versioned routes. Prefer route lists over the predicate form, which skips path normalization. https://docs.nestjs.com/security/csrf#excluding-routes
- Limits: GET handlers must not change state, WebSocket upgrades are not checked (validate the handshake `Origin` if gateways use cookies), a proxy rewriting `Host` makes old browsers get 403, and a 403 carries no CORS headers when CORS uses the `cors` create option. https://docs.nestjs.com/security/csrf#limitations
- Rejections are a `ForbiddenException` seen only by global filters. https://docs.nestjs.com/security/csrf#rejected-requests

## CORS

- `enableCors()` uses Express `cors` or `@fastify/cors`, whose default methods differ: Fastify allows only `GET,HEAD,POST`. List methods explicitly: https://docs.nestjs.com/security/cors#default-allowed-methods
  ```typescript
  app.enableCors({ methods: ['GET', 'HEAD', 'PUT', 'PATCH', 'POST', 'DELETE'] });
  ```

## Rate limiting

- After importing `ThrottlerModule.forRoot()`, choose how to bind `ThrottlerGuard`; to bind it globally, add the provider `{ provide: APP_GUARD, useClass: ThrottlerGuard }`. `ttl` and `blockDuration` are milliseconds (use `seconds()`, `minutes()`). https://docs.nestjs.com/security/rate-limiting
  ```typescript
  ThrottlerModule.forRoot([{ name: 'short', ttl: 1000, limit: 3 }, { name: 'long', ttl: 60000, limit: 100 }])
  ```
- `@Throttle({ default: { limit, ttl } })` and `@SkipThrottle({ default: true })` take objects keyed by throttler name. https://docs.nestjs.com/security/rate-limiting#customization
- Storage is per-process memory. Use shared storage (Redis) when running several instances. https://docs.nestjs.com/security/rate-limiting#storages
- Behind a proxy trust it: `app.set('trust proxy', 'loopback')` (Express), `new FastifyAdapter({ trustProxy: 'loopback' })`, `new NodeAdapter({ trustProxy: 'loopback' })`. A number as `trustProxy` trusts no proxy on Fastify v5.12.1+ and the Node adapter, so every client counts as one. https://docs.nestjs.com/security/rate-limiting#proxies
- The `req.ips[0]` tracker override is for Express and the Node adapter only. On Fastify `ips[0]` is the proxy: keep the default tracker. https://docs.nestjs.com/security/rate-limiting#proxies
- IPv6 clients are grouped by `ipv6SubnetPrefix` (default 64). https://docs.nestjs.com/security/rate-limiting#configuration
- WebSocket and GraphQL need `ThrottlerGuard` subclasses (`handleRequest`, `getRequestResponse`). Do not register the WS guard globally. https://docs.nestjs.com/security/rate-limiting#websockets
- A `ThrottlerGuard` registered as `APP_GUARD` in the root module runs before the `@nestjs/authentication` guard (global guards run in module order) and before validation, so a custom tracker must not trust the body shape. https://docs.nestjs.com/security/authentication#rate-limit-sign-in

## Encryption and password hashing

- Nest wraps no encryption API: use `node:crypto` (`createCipheriv`, `scrypt`). https://docs.nestjs.com/security/encryption-and-hashing#encryption
- Default for passwords: `PasswordHasher` from `@nestjs/authentication` (scrypt, OWASP parameters, about 128 MiB per hash, on the libuv pool, self-describing hash string). https://docs.nestjs.com/security/encryption-and-hashing#hashing
  ```typescript
  const ok = await this.passwordHasher.verify(password, found?.passwordHash);
  if (ok && this.passwordHasher.needsRehash(found.passwordHash)) { /* store a new hash */ }
  ```
- Pass `undefined` for an unknown user so `verify()` checks a dummy hash in the same time. Empty passwords and passwords over 4 KiB are refused. Use `new PasswordHasher({ logN: 10 })` in tests. https://docs.nestjs.com/security/encryption-and-hashing#hashing
- Escape hatch: `argon2` or `bcrypt` through your own provider. Both are native modules, and bcrypt reads only the first 72 bytes. https://docs.nestjs.com/security/encryption-and-hashing#hashing

## Old patterns

| Old | Current |
| --- | --- |
| `app.use(helmet())` / `app.register(@fastify/helmet)` | `app.useSecurityHeaders()` |
| `csrf-csrf` or `@fastify/csrf-protection` tokens | `app.enableCsrfProtection()` (keep tokens only for very old browsers) |
| Throttler `ttl` in seconds, positional `@Throttle()` | `ttl` in ms, `@Throttle({ name: {...} })` |

https://docs.nestjs.com/security/helmet#using-helmet-directly https://docs.nestjs.com/security/csrf#when-you-need-a-token-based-approach https://docs.nestjs.com/security/rate-limiting#migration-guide

- Kept Helmet must be registered before other `app.use()` calls. https://docs.nestjs.com/security/helmet#using-helmet-directly
