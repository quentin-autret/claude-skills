# Authentication (@nestjs/authentication)

## Model

- `AuthenticationModule.forRoot()` registers a global guard: every route then requires a user and answers 401. Open routes with `@Public()` on class or method. `globalGuard: false` leaves routes open until `@UseGuards(AuthenticationGuard)`. https://docs.nestjs.com/security/authentication#module-options
- The package never reads your users table. A credential provider (extends `SessionCookieProvider<User>`, `JwtBearerProvider<User>` or `ApiKeyProvider<User>`) registers itself in its constructor with `registry.registerProvider(this)` and loads the user in `validate()` (`findKey(id)` for `ApiKeyProvider`), which runs on every request. Returning `null` rejects. https://docs.nestjs.com/security/authentication#register-the-authentication-module
- Several providers are tried by ascending `order` (`registerProvider(this, { order: 1 })`). Equal orders fail at startup. https://docs.nestjs.com/security/authentication#issue-tokens-to-the-mobile-app
- `forRoot()` takes plain values only and is evaluated at import time. Use `forRootAsync()` for env or `ConfigService` values. https://docs.nestjs.com/security/authentication#verify-email-addresses
- Durations take ms or strings (`'15m'`, `'14d'`). `@nestjs/jwt` counts seconds, so a numeric `3600` here is 3.6 s (a startup warning flags lifetimes under a minute). https://docs.nestjs.com/security/authentication#register-the-authentication-module
- Each feature needs both its option and its handler provider (for example `emailVerification: { url }` plus an `EmailVerificationHandler`). One without the other, or an out-of-range value such as `Number()` of an unset variable, fails at startup. https://docs.nestjs.com/security/authentication#module-options
- `@Authenticate({ optional, mfa, verifiedEmail, providers })`: `optional` lets guests through as `null`, `verifiedEmail` answers 403 `email_unverified`, `providers: [SessionAuth]` accepts only those credentials. https://docs.nestjs.com/security/authentication#route-options
- `AuthenticationContext` (singleton on `AsyncLocalStorage`) is empty in middleware, guards, filters, queue consumers and cron jobs. Use `run()` to act as a user. https://docs.nestjs.com/security/authentication#read-the-current-user
- DTOs on sign-up and sign-in routes are enforced only by a global validation pipe. https://docs.nestjs.com/security/authentication#add-sign-up-and-sign-in

## Sessions (cookies)

- `SignInService.signIn(userId, { method: 'password' })` creates the session and sets the `__Host-sid` cookie (HttpOnly, Secure, SameSite=Lax) on Express and Fastify, no `@Res()` needed. It ends the previous session and refuses cross-site sign-in posts. `session.mfa === 'pending'` means a second factor is due. https://docs.nestjs.com/security/authentication#add-sign-up-and-sign-in
- The cookie is ignored on cross-origin writes and WebSocket handshakes. List other web origins in `session.trustedOrigins`. On Nest 12.1 or later, also call `app.enableCsrfProtection()` with the same origins. https://docs.nestjs.com/security/authentication#add-sign-up-and-sign-in
- Browsers accept `Secure` cookies on `http://localhost`. On another plain-HTTP host set `session.cookie.secure: false` (cookie then named `sid`). https://docs.nestjs.com/security/authentication#add-sign-up-and-sign-in
- Defaults: `absoluteTtl` `'7d'`, `idleTtl` `'1d'`. https://docs.nestjs.com/security/authentication#module-options
- Rotate on privilege changes with `SignInService.rotateSession()`. After a password or MFA change also `SessionService.revokeAll(userId, { except: current.id })` and `TokenService.revokeAll(userId)`. A password reset does this itself. https://docs.nestjs.com/security/authentication#production-checklist
- Make sign-out `@Public()` so a pending session can sign out. https://docs.nestjs.com/security/authentication#let-customers-manage-their-sessions

## Tokens (JWT access, opaque refresh)

```typescript
accessToken: { key: process.env.JWT_SECRET!, issuer: 'https://api.example.com', audience: 'mobile-app', ttl: '15m' }
```
https://docs.nestjs.com/security/authentication#issue-tokens-to-the-mobile-app

- HS256 needs a key of at least 32 bytes. Use ES256 when other services verify. For another issuer pass `jwks` plus both `issuer` and `audience`. An empty key or jwks (unset env var) fails at startup. https://docs.nestjs.com/security/authentication#issue-tokens-to-the-mobile-app
- Refresh tokens are opaque, hashed and single use. Reusing a spent one revokes the whole family. Keep changing facts (roles) out of `claims`: they persist across refreshes. https://docs.nestjs.com/security/authentication#issue-tokens-to-the-mobile-app

## Two-factor, magic links, OIDC, API keys

- TOTP requires `mfa.encryption.keys` (startup error otherwise). Without `mfa` configured, 2FA is off for everyone, including enrolled users, and every app sharing the user database must configure the same features. https://docs.nestjs.com/security/authentication#production-checklist
- After enrollment, sessions start pending and routes answer 401 `mfa_required`. The verify route must be `@Public()`. https://docs.nestjs.com/security/authentication#add-two-factor-authentication
- Emailed links (verification, reset, magic link) must open a page that POSTs the token back: scanners follow GET links and would spend it. https://docs.nestjs.com/security/authentication#production-checklist
- Do not `await passwordResetService.request(email)`: it answers the same 202 in the same time for unknown addresses. https://docs.nestjs.com/security/authentication#reset-forgotten-passwords
- OIDC: link identities by (provider, subject), not email. Link by email only if both the provider and your account verified it (account pre-hijacking). `microsoft()` needs a tenant GUID. https://docs.nestjs.com/security/authentication#add-sign-in-with-google
- API keys (`<prefix>_<id>_<secret>`): store id and hash only, show the key once. Restrict key management with `@Authenticate({ providers: [SessionAuth] })`. Keys never satisfy MFA, emit no events, and are not revoked by a password reset. https://docs.nestjs.com/security/authentication#issue-api-keys-to-partners
- The package does not rate-limit. Use `@nestjs/throttler` with a per-IP and a per-account-plus-IP throttler (`normalizeIp()`), and `trust proxy` behind a proxy. https://docs.nestjs.com/security/authentication#rate-limit-sign-in

## Stores and production

- Stores are in memory by default. With `NODE_ENV=production` startup fails while a store needed by a configured feature is missing (`allowInMemoryStorage: true` overrides). https://docs.nestjs.com/security/authentication#the-store-contract
- A store is a provider calling `storage.registerSource({ sessions: this, refreshTokens: this, ... })` in its constructor. Every "only once" operation must be one conditional statement (`UPDATE ... WHERE used_at IS NULL`, `DELETE ... RETURNING`), never read-then-write. With Prisma use `updateMany()`/`deleteMany()` and never `upsert()` for `touchSession()`. https://docs.nestjs.com/security/authentication#the-store-contract With TypeORM never `save()`. https://docs.nestjs.com/security/authentication#keep-sessions-and-tokens-in-your-database
- Verify stores with `authenticationStoreContract(factory, { concurrent: true })` on a real PostgreSQL pool. https://docs.nestjs.com/security/authentication#keep-sessions-and-tokens-in-your-database
- Size servers for scrypt (128 MiB per hash, 4 libuv threads unless `UV_THREADPOOL_SIZE`). https://docs.nestjs.com/security/authentication#production-checklist
- Import `AuthenticationModule` before `AuthorizationModule`. https://docs.nestjs.com/security/authentication#production-checklist

## GraphQL, WebSockets, events, testing

- GraphQL `context` must be a function returning `{ req }`: `context: ({ req, res, extra }) => ({ req: req ?? extra?.request, res })`. Add `fieldResolverEnhancers: ['guards', 'interceptors']` to guard field resolvers and give them the context. https://docs.nestjs.com/security/authentication#graphql-and-websockets
- No guard runs on `handleConnection()`: call `wsAuthenticator.authenticateConnection(client, request)` there. Read `@CurrentUser()`, not `client.user` or `req.user` (last message wins). https://docs.nestjs.com/security/authentication#graphql-and-websockets
- `AuthenticationError` maps to HTTP, GraphQL `UNAUTHENTICATED`, `WsException` or `RpcException`. Events on `AuthenticationEvents.events$` and `nestjs:authentication:<type>`. https://docs.nestjs.com/security/authentication#errors
- Tests: override `PasswordHasher` with `new PasswordHasher({ logN: 10 })`, sign users in with `SessionService.create()` or `TokenService.issue()`. https://docs.nestjs.com/security/authentication#testing

## Old patterns: Passport

The Passport recipe covers `@nestjs/passport`. https://docs.nestjs.com/recipes/passport

- Wrap strategies in named guard classes (`class LocalAuthGuard extends AuthGuard('local') {}`) instead of magic strings. https://docs.nestjs.com/recipes/passport#login-route
- Strategies cannot be request-scoped: use `passReqToCallback: true` and `moduleRef.resolve(AuthService, ContextIdFactory.getByRequest(request))`. https://docs.nestjs.com/recipes/passport#request-scoped-strategies
- Since Passport 0.6, `req.logout()` takes a callback: wrap it in a Promise. https://docs.nestjs.com/recipes/passport#logout-route
- GraphQL: override `getRequest()` with `GqlExecutionContext.create(context).getContext().req`. https://docs.nestjs.com/recipes/passport#graphql
- A Passport `AuthGuard()` subclass carries no ordering marker, so a wrong order only logs an error. Register it with `APP_GUARD`, never `app.useGlobalGuards()` (runs after every `APP_GUARD`). https://docs.nestjs.com/security/authorization#register-the-authorization-module
