# Authorization (@nestjs/authorization)

## Contents
- Model and registration
- Where each check goes
- Route parameter checks
- before() hooks
- 401 versus 403 and errors
- Production and testing

## Model and registration

- Permissions are `@Policy()` classes. Each method whose first parameter is the user and which returns `boolean` or `Promise<boolean>` is an ability. Only a literal `true` allows. Checks are type-checked (a misspelled ability does not compile). https://docs.nestjs.com/security/authorization#write-a-policy-for-products
  ```typescript
  AuthenticationModule.forRootAsync({ useFactory: () => ({ accessToken: { /* ... */ } }) }),
  AuthorizationModule.forRoot(),
  ```
  https://docs.nestjs.com/security/authorization#register-the-authorization-module
- The guard denies by default: a handler runs only if it declares `@Can()`, `@Can.Anyone()` or `@Public()`. A forgotten declaration closes the route. Startup logs every denied handler. https://docs.nestjs.com/security/authorization#register-the-authorization-module
- Import `AuthenticationModule` before `AuthorizationModule`: global guards run in module scan order. With `@nestjs/authentication`, the wrong order fails startup. With another guard (Passport) it only logs an error and every check sees a guest. https://docs.nestjs.com/security/authorization#register-the-authorization-module
- Register other authentication guards with `APP_GUARD` as singletons. A request-scoped `APP_GUARD` and `app.useGlobalGuards()` always run after the authorization guard. https://docs.nestjs.com/security/authorization#production-checklist
- The guard reads `request.user` (and transport equivalents). Use `getUser` for another location. With `@UseGuards()` authentication, pass `globalGuard: false` and list `AuthorizationGuard` after it. https://docs.nestjs.com/security/authorization#register-the-authorization-module
- Policies are discovered as providers. An unregistered policy in `@Can()` fails startup. https://docs.nestjs.com/security/authorization#write-a-policy-for-products
- Policies must be singletons: request-scoped or transient ones throw at startup. https://docs.nestjs.com/security/authorization#inject-a-service-into-a-policy

## Where each check goes

- Route level `@Can(Policy, 'create')` for checks that do not need a record. Only abilities whose user parameter accepts `null` fit there, since the guard cannot know whether a route has a user. An ability that also takes a record does not compile on a route. https://docs.nestjs.com/security/authorization#write-a-policy-for-products
- Record checks belong in the service after loading: `authorize()` throws on denial, `can()` resolves a boolean (for flags such as `canRefund`). Routes that check in the service declare `@Can.Anyone()`. https://docs.nestjs.com/security/authorization#tell-guests-from-forbidden-users-and-expose-permissions
  ```typescript
  @Get(':id')
  @Authenticate({ optional: true })
  @Can.Anyone()
  findOne(@CurrentUser() user: User | null, @Param('id') id: string) {}
  ```
- Filter lists in the query with an ability the query can use (`viewAll`), not by checking each loaded row (breaks pagination). https://docs.nestjs.com/security/authorization#check-orders-record-by-record
- Guests on a route are authentication's call: `@Authenticate({ optional: true })` gives `null`. `@Public()` ignores tokens entirely, so a signed-in user browses as a guest. https://docs.nestjs.com/security/authorization#write-a-policy-for-products
- A 403 confirms the record exists. When ids are guessable, check with `can()` and throw `NotFoundException`. https://docs.nestjs.com/security/authorization#check-orders-record-by-record

## Route parameter checks

```typescript
@Can(OrderPolicy, 'viewOrdersOf', (context) => [context.switchToHttp().getRequest().params.customerId])
```
https://docs.nestjs.com/security/authorization#check-a-route-parameter

- The function runs in the guard before any pipe: values are raw strings, and an invalid id the user may not see gets 403, not 400. https://docs.nestjs.com/security/authorization#check-a-route-parameter
- Pass ids, not records (loading in the guard loads twice and answers 403 before a 404). A standalone helper returns an array: end it with `as const`. https://docs.nestjs.com/security/authorization#check-a-route-parameter
- On WebSockets and microservices throw `WsException` or `RpcException` from that function. https://docs.nestjs.com/security/authorization#check-a-route-parameter

## before() hooks

- `before(user: User | null)` runs ahead of every ability of its policy. Return `true` to allow, `false` to deny, nothing to let the ability decide. https://docs.nestjs.com/security/authorization#let-admins-through-with-before
- It also skips rules that must bind everyone (nobody refunds a shipped order): check the ability name and return nothing for those, typed with `PolicyBefore<User, Ability<OrderPolicy>>`. https://docs.nestjs.com/security/authorization#let-admins-through-with-before

## 401 versus 403 and errors

- Guest on a protected route: 401 from authentication. Guest denied by a policy on an optional route: 401. Signed-in user denied: 403. The body never names the ability. https://docs.nestjs.com/security/authorization#tell-guests-from-forbidden-users-and-expose-permissions
- `authorize()` throws `AuthorizationError` (`reason`, `status`, `policy`, `ability`), mapped when it leaves a handler. Thrown from your own guard or middleware it becomes a 500: use `can()` there. https://docs.nestjs.com/security/authorization#errors
- Over `graphql-ws` the client gets only the message, without `extensions.code`. https://docs.nestjs.com/security/authorization#errors

## Production and testing

- Where Nest runs no global guard nothing is denied: GraphQL field resolvers without `fieldResolverEnhancers: ['guards']`, and hybrid-app message handlers without `inheritAppConfig: true`. https://docs.nestjs.com/security/authorization#what-a-route-declares
- On a method, `@Can.Anyone()` or `@Public()` lifts the class `@Can()`. On a class, neither lifts a method's `@Can()`. https://docs.nestjs.com/security/authorization#what-a-route-declares
- Do not cache record-checked routes by URL: `CacheInterceptor` answers before the handler and would serve one user's record to another. https://docs.nestjs.com/security/authorization#production-checklist
- Re-check state at write time with a conditional update: two concurrent requests can both pass the check. https://docs.nestjs.com/security/authorization#production-checklist
- Audit denials through `AuthorizationEvents.events$` (`can()` returning false emits nothing). https://docs.nestjs.com/security/authorization#events
- Unit-test policies with `new OrderPolicy(dep)`. Calling an ability directly skips `before()`. Outside a handler a denial stays an `AuthorizationError`. Replace a policy with `overrideProvider()`. https://docs.nestjs.com/security/authorization#testing
