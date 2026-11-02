# WebSockets

## Contents
- Gateways
- Lifecycle and authentication
- Exceptions, pipes, guards, interceptors
- Adapters and scaling

## Gateways

- Gateways are instantiated only when listed in a module's `providers`. They share the HTTP port unless one is given: `@WebSocketGateway(81, { namespace: 'events' })`. https://docs.nestjs.com/websockets/gateways#overview
- Use `@MessageBody()` and `@ConnectedSocket()`. The decorator-less signature forces socket mocks in tests. https://docs.nestjs.com/websockets/gateways#overview
- Returning nothing, `null` or `undefined` sends no reply, but `false` and `0` are sent. The client receives the reply only with an ack callback. With `@Ack()` Nest sends nothing from the return value: call `ack()` yourself. https://docs.nestjs.com/websockets/gateways#overview
- `client.emit()` bypasses interceptors, which only see the return value. Several replies: return `{ event, data }` (https://docs.nestjs.com/websockets/gateways#multiple-responses) or an Observable (https://docs.nestjs.com/websockets/gateways#asynchronous-responses).
- With `ClassSerializerInterceptor`, return a class instance implementing `WsResponse`, plain objects are ignored. https://docs.nestjs.com/websockets/gateways#multiple-responses
- `@WebSocketServer()` injects a `Namespace` instead of a `Server` when `namespace` is set. https://docs.nestjs.com/websockets/gateways#server-and-namespace
- New in v12: request-scoped providers in gateways, one instance per connected socket, with the socket under `@Inject(REQUEST)`. Keep singletons unless you need per-connection state. https://docs.nestjs.com/websockets/gateways#request-scoped-gateways
  ```typescript
  @Injectable({ scope: Scope.REQUEST })
  export class ConnectionStateService {
    constructor(@Inject(REQUEST) private readonly client: Socket) {}
  }
  ```

## Lifecycle and authentication

- Nest does not wait for an async `handleConnection()` before handling messages, so it is not a gate. Check each message with guards, and reject connections in connection middleware: socket.io `server.use()` in `afterInit()`, or `verifyClient` in the gateway options with `WsAdapter`. https://docs.nestjs.com/websockets/gateways#lifecycle-hooks
- WebSocket upgrades are not checked by `enableCsrfProtection()`: validate the handshake `Origin` if gateways authenticate with cookies. https://docs.nestjs.com/security/csrf#limitations

## Exceptions, pipes, guards, interceptors

- Throw `WsException`. Any other exception (including `BadRequestException` from validation pipes) reaches the client as `Internal server error`. https://docs.nestjs.com/websockets/pipes
  ```typescript
  @UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new WsException(errors) }))
  ```
- Global filters (`useGlobalFilters()`, `APP_FILTER`) do not apply to gateways: bind `@UseFilters()` on the gateway or method. https://docs.nestjs.com/websockets/exception-filters#filters Global pipes, guards and interceptors do apply. https://docs.nestjs.com/websockets/pipes#binding-pipes https://docs.nestjs.com/websockets/guards#binding-guards https://docs.nestjs.com/websockets/interceptors
- Method, gateway and global pipes run on every handler parameter: bind validation on `@MessageBody()` itself. `@MessageBody({ schema })` works with `StandardSchemaValidationPipe`. https://docs.nestjs.com/websockets/pipes#schema-based-validation
- A guard returning `false` throws `WsException('Forbidden resource')`. https://docs.nestjs.com/websockets/guards
- `BaseWsExceptionFilter` accepts `includeCause: false` to omit `cause`. https://docs.nestjs.com/websockets/exception-filters#inheritance
- Microservice pre-request hooks do not apply to gateways. https://docs.nestjs.com/microservices/pre-request-hooks#logging-and-metrics

## Adapters and scaling

- Several socket.io instances need a Redis adapter (extend `IoAdapter`, `createAdapter(pub, sub)`) and either `transports: ['websocket']` on the client or sticky sessions. Redis alone is not enough. https://docs.nestjs.com/websockets/adapter#extend-socketio
- `WsAdapter` (`@nestjs/platform-ws`) is faster with fewer features. It has no namespaces and throws if `namespace` is set: use `path`. Messages are `{ event, data }` unless you pass a `messageParser`. https://docs.nestjs.com/websockets/adapter#ws-library
- Use the built-in `WsAdapter` rather than a custom adapter. https://docs.nestjs.com/websockets/adapter#advanced-custom-adapter
- WebSocket rate limiting needs a `ThrottlerGuard` subclass that overrides `handleRequest()`, not registered globally with `APP_GUARD` or `useGlobalGuards()`. https://docs.nestjs.com/security/rate-limiting#websockets
