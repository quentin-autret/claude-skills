# Microservices: transports, patterns, enhancers, hybrid apps

## Contents
- Clients and patterns
- Redis, MQTT, NATS, RabbitMQ
- Kafka
- gRPC
- Enhancers and pre-request hooks
- Custom transporters
- Hybrid applications
- Old patterns

## Clients and patterns

- `@MessagePattern()` works only in controllers. The runtime ignores it in providers. https://docs.nestjs.com/microservices/basics#request-response
- Request-response needs a reply channel that most transports create manually, with overhead. Default to `@EventPattern()` when no reply is needed. Several event handlers for one pattern all run in parallel. https://docs.nestjs.com/microservices/basics#event-based
- Register clients with `ClientsModule.register()`/`registerAsync()` and inject by token, not with the `@Client()` decorator (harder to test and share). https://docs.nestjs.com/microservices/basics#client-producer-class
- `ClientProxy` connects lazily on the first call. To fail at bootstrap instead, `await this.client.connect()` in `onApplicationBootstrap()`. https://docs.nestjs.com/microservices/basics#client-producer-class
- `send()` is cold (nothing is sent until subscribed). `emit()` is hot (sent immediately). https://docs.nestjs.com/microservices/basics#sending-messages
- To build transport options from injected providers, pass `AsyncMicroserviceOptions` (`useFactory`, `inject`) to `createMicroservice()`. https://docs.nestjs.com/microservices/basics#dynamic-configuration
- With the TCP transporter (NestJS v12.1.1+), client `on()` listeners stay registered until `close()` and are re-attached to the next connection, without deduplication: register each listener once, never after a reconnect. https://docs.nestjs.com/microservices/basics#listening-to-internal-events

## Redis, MQTT, NATS, RabbitMQ

- Redis uses Pub/Sub: a message with no subscriber is lost. `wildcards: true` must be set on both server and client. `unwrap()` returns a tuple `[pub, sub]`. https://docs.nestjs.com/microservices/redis#wildcards https://docs.nestjs.com/microservices/redis#underlying-driver-access
- MQTT subscriptions default to QoS 0. Raise globally with `subscribeOptions: { qos: 2 }` or per pattern with `@EventPattern('critical-events', { qos: 2 })`. https://docs.nestjs.com/microservices/mqtt#quality-of-service-qos
- NATS targets NATS v3 through `@nats-io/transport-node`. Load-balance with `queue: 'cats_queue'`. Record headers override client `headers`. https://docs.nestjs.com/microservices/nats#queue-groups https://docs.nestjs.com/microservices/nats#record-builders
- RabbitMQ `noAck` defaults to `true`. For acknowledgements set `noAck: false` and ack every message, or it is re-queued when the consumer dies: https://docs.nestjs.com/microservices/rabbitmq#message-acknowledgement
  ```typescript
  context.getChannelRef().ack(context.getMessage());
  ```

## Kafka

- The client type is `ClientKafkaProxy`. Request-response needs `subscribeToResponseOf()` for each pattern, before `connect()`: https://docs.nestjs.com/microservices/kafka#message-response-subscription
  ```typescript
  async onModuleInit() {
    this.client.subscribeToResponseOf('hero.kill.dragon');
    await this.client.connect();
  }
  ```
- You need at least one reply topic partition per running Nest app (4 apps, 3 partitions: 1 app fails to send). The reply topic is the request topic plus `.reply`. https://docs.nestjs.com/microservices/kafka#message-pattern https://docs.nestjs.com/microservices/kafka#naming-conventions
- `-server`/`-client` is appended to `clientId` and `groupId`. Change with `postfixId`. https://docs.nestjs.com/microservices/kafka#naming-conventions
- Patterns accept a `RegExp` on Kafka only (`@EventPattern(/^hero\..+$/)`), read the source with `context.getTopic()`. https://docs.nestjs.com/microservices/kafka#regular-expression-patterns
- Every unhandled exception in an event handler is retriable (offset not committed, redelivered). In message handlers throw `KafkaRetriableException` to retry. https://docs.nestjs.com/microservices/kafka#retriable-exceptions
- A message counts as a request only with the `kafka_correlationId` and `kafka_replyTopic` headers `send()` adds. Messages from other producers are events even in `@MessagePattern()`: return completing streams and handle bad messages instead of throwing. https://docs.nestjs.com/microservices/kafka#retriable-exceptions
- Messages over 16 MiB after decompression are skipped with their offset committed. Raise `parser.maxMessageSize`. https://docs.nestjs.com/microservices/kafka#incoming
- Slow handlers call `await context.getHeartbeat()()` between steps. Manual commit is offset + 1 with `run: { autoCommit: false }`. https://docs.nestjs.com/microservices/kafka#context https://docs.nestjs.com/microservices/kafka#commit-offsets

## gRPC

- Copy protos to `dist` in `nest-cli.json`: `"assets": ["**/*.proto"], "watchAssets": true`. Default `url` is `localhost:5000` (use `0.0.0.0:50051` in Docker) and default credentials are insecure. https://docs.nestjs.com/microservices/grpc#options
- Use `@GrpcMethod('HeroesService', 'FindOne')`, not `@MessagePattern()`. Without arguments, the service is the class name and the rpc is the capitalized method name. https://docs.nestjs.com/microservices/grpc#sample-grpc-service
- Clients inject `ClientGrpc` and call `getService<HeroesService>('HeroesService')` in `onModuleInit()`. Methods are lower camel case. Fields with an underscore are not sent unless `loader: { keepCase: true }`. https://docs.nestjs.com/microservices/grpc#client
- Throw status-specific exceptions (`GrpcNotFoundException`, `GrpcInvalidArgumentException`, or `GrpcException(message, GrpcStatus.X)`) and register the filter, or nothing is serialized into gRPC errors: https://docs.nestjs.com/microservices/grpc#exception-handling
  ```typescript
  app.useGlobalFilters(new GrpcExceptionFilter());
  ```
- `@GrpcStreamMethod()` receives an `Observable` and must return one. Client-streaming rpcs cannot use `@GrpcMethod()`. https://docs.nestjs.com/microservices/grpc#grpc-streaming

## Enhancers and pre-request hooks

- Throw `RpcException`, not `HttpException`: unless a custom exception filter handles it, an `HttpException` reaches the client as a generic `Internal server error`. Give validation pipes an `exceptionFactory`: https://docs.nestjs.com/microservices/pipes#binding-pipes
  ```typescript
  @UsePipes(new ValidationPipe({ exceptionFactory: (errors) => new RpcException(errors) }))
  ```
- An RPC filter's `catch()` returns an `Observable` (`throwError(() => exception.getError())`). Extend `BaseRpcExceptionFilter` to reuse the default. https://docs.nestjs.com/microservices/exception-filters#filters
- An error a filter rethrows for an `@EventPattern()` handler never reaches the producer: handle it inside the filter. https://docs.nestjs.com/microservices/exception-filters
- Pre-request hooks are the microservice equivalent of middleware: they run before guards, interceptors and pipes, are global, and their exceptions reach exception filters. Register before `listen()`/`init()` (later ones are ignored with a warning), and always call `next()`: https://docs.nestjs.com/microservices/pre-request-hooks#asynclocalstorage-propagation
  ```typescript
  app.registerPreRequestHook((ctx, next) => new Observable((subscriber) => {
    als.run({ correlationId: randomUUID() }, () => { next().subscribe(subscriber); });
  }));
  ```

## Custom transporters

- A server extends `Server implements CustomTransportStrategy` (`listen()`, `close()`) and is passed as `{ strategy: new X() }`. Handlers in `this.messageHandlers` may return streams: subscribe when `isObservable(result)`: with interceptors the handlers are wrapped in RxJS streams, and their logic only runs once subscribed. https://docs.nestjs.com/microservices/custom-transport#creating-a-strategy
- A client extends `ClientProxy` (`connect`, `close`, `dispatchEvent`, `publish`, `unwrap`) and calls `callback({ response, isDisposed: true })`. Register with `customClass` (the class, not an instance). https://docs.nestjs.com/microservices/custom-transport#client-proxy https://docs.nestjs.com/microservices/custom-transport#message-serialization

## Hybrid applications

- Use `app.connectMicroservice()` per transport. Call `app.listen()`/`app.init()` before `startAllMicroservices()`, otherwise handlers consume messages before `onModuleInit`/`onApplicationBootstrap` complete. https://docs.nestjs.com/faq/hybrid-application
- Connected microservices inherit no global pipes, guards, interceptors or filters (`useGlobal*()` or `APP_*`). Opt in, with `useGlobal*()` called before `connectMicroservice()`: https://docs.nestjs.com/faq/hybrid-application#sharing-configuration
  ```typescript
  app.useGlobalPipes(new ValidationPipe());
  app.connectMicroservice(opts, { inheritAppConfig: true });
  ```
- `connectMicroservice()` initializes immediately, so pre-request hooks registered on it afterwards are skipped unless `deferInitialization: true`. https://docs.nestjs.com/microservices/pre-request-hooks#binding-hooks
- Bind a pattern to one transport with a second argument: `@MessagePattern('time.us.*', Transport.NATS)`. https://docs.nestjs.com/faq/hybrid-application

## Old patterns

| Old | Current |
| --- | --- |
| `nats` package | `@nats-io/transport-node` (NATS v3) |
| `JSON.parse(new TextDecoder().decode(msg))` in NATS deserializers | `msg.json()` on the full message |
| Plain `RpcException` over gRPC (always `UNKNOWN`) | `Grpc*Exception` + `GrpcExceptionFilter` |
| `@Client()` property | `ClientsModule.register()` + `@Inject()` |

https://docs.nestjs.com/microservices/nats#installation https://docs.nestjs.com/microservices/nats#custom-serializers-and-deserializers https://docs.nestjs.com/microservices/grpc#exception-handling https://docs.nestjs.com/microservices/basics#client-producer-class
