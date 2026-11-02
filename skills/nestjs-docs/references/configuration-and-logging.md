# Configuration and logging

## Contents
- ConfigModule
- Env file precedence
- Validating the environment
- Built-in logger
- Structured params and JSON output
- Custom logger with DI and buffered logs

## ConfigModule

```typescript
ConfigModule.forRoot({ isGlobal: true, load: [configuration], cache: true })
```
https://docs.nestjs.com/application/configuration#use-module-globally

- Read required values with `getOrThrow<T>('KEY')`. Type the service as `ConfigService<Env, true>` with `get('PORT', { infer: true })` to drop `undefined`. https://docs.nestjs.com/application/configuration#using-the-configservice
- Namespaces: `registerAs('database', () => ({ ... }))`, inject with `@Inject(databaseConfig.KEY) cfg: ConfigType<typeof databaseConfig>`, or plug into another module: `TypeOrmModule.forRootAsync(databaseConfig.asProvider())`. https://docs.nestjs.com/application/configuration#configuration-namespaces
- `forFeature()` values are registered during module init, whose order is indeterminate: read them in `onModuleInit()`, not a constructor. https://docs.nestjs.com/application/configuration#partial-registration
- `cache: true` speeds up `get()` for `process.env` values. https://docs.nestjs.com/application/configuration#cache-environment-variables
- `ConditionalModule.registerWhen(FooModule, 'USE_FOO')` loads the module unless the variable is `false`. An undefined variable loads it. It throws after 5 s if env loading has not finished. https://docs.nestjs.com/application/configuration#conditional-module-configuration
- Module `forRoot()` arguments are evaluated when the file is imported, so `process.env` must already be populated. Use `forRootAsync()` with `inject: [ConfigService]` for values from configuration. The docs state this for the Drizzle and authentication modules. https://docs.nestjs.com/data/drizzle#registering-the-database https://docs.nestjs.com/security/authentication#verify-email-addresses
- For values needed before `ConfigModule` loads (such as `createMicroservice()` options), use `nest start --env-file .env`. https://docs.nestjs.com/application/configuration#getting-started
- The Nest CLI does not copy YAML or other non-TS assets to `dist`: list them in `nest-cli.json` `compilerOptions.assets`. https://docs.nestjs.com/application/configuration#custom-configuration-files

## Env file precedence

- `.env` is read from the current working directory. A runtime variable beats the file unless `override: true`. https://docs.nestjs.com/application/configuration#getting-started
- With several `envFilePath` entries the first file wins. With Node's `--env-file` the last file wins and runtime variables always win. https://docs.nestjs.com/application/configuration#native-env-support
- With native loading set `ignoreEnvFile: true`. `process.loadEnvFile()` must run in a file imported before `AppModule`, a call in the body of `main.ts` is too late. https://docs.nestjs.com/application/configuration#native-env-support
- Node's `util.parseEnv` is not fully dotenv compatible (BOM, quotes, behavior changes across Node minors) and does not expand variables: do not combine `parser: parseEnv` with `expandVariables`. https://docs.nestjs.com/application/configuration#native-env-support

## Validating the environment

- `validationSchema` accepts any Standard Schema. The docs recommend Zod or another Standard Schema library for new projects: https://docs.nestjs.com/application/configuration#schema-validation
  ```typescript
  validationSchema: z.object({
    NODE_ENV: z.enum(['development', 'production', 'test', 'provision']).default('development'),
    PORT: z.coerce.number().default(3000),
  }),
  ```
- Values are strings: coerce them. `ConfigService` serves the schema output. A variable without default makes bootstrap throw. Undeclared variables (PATH, HOME) stay available and do not fail validation. https://docs.nestjs.com/application/configuration#schema-validation
- Joi needs v18 or later. Library settings live under `validationOptions.libraryOptions`. https://docs.nestjs.com/application/configuration#schema-validation
- `load` files are not validated by `validationSchema`: validate inside the factory. A custom `validate()` must be synchronous. https://docs.nestjs.com/application/configuration#custom-validate-function

## Built-in logger

- One logger per class: `private readonly logger = new Logger(MyService.name)`. https://docs.nestjs.com/application/logger#using-the-logger-for-application-logging
- Levels cascade from the most severe listed: `['log']` enables warn, error, fatal too. `['debug', 'error']` enables debug, error, fatal but not log or warn. https://docs.nestjs.com/application/logger#basic-customization
- Levels from the environment: `logger: filterLogLevels(process.env.LOG_LEVEL ?? '>=warn')`. Change later with `app.useLogger(['error', 'warn'])`. https://docs.nestjs.com/application/logger#basic-customization
- Expensive messages: pass a function, `this.logger.debug(() => ...)`, evaluated only when the level is on. https://docs.nestjs.com/application/logger#using-the-logger-for-application-logging
- The `+5ms` diff is relative to the previous line from any `ConsoleLogger` in the process. https://docs.nestjs.com/application/logger#logs-with-timestamps
- Reach for Pino or Winston only for in-process transports, redaction, custom serializers or maximum throughput. https://docs.nestjs.com/application/logger#use-external-logger

## Structured params and JSON output

- In v12, plain objects after the message are attached to the same entry instead of emitted as separate lines. `structuredParams: false` restores v11 behavior. https://docs.nestjs.com/application/logger#structured-logging-params
  ```typescript
  this.logger.log('Payment captured', { orderId: order.id, amount: order.total });
  ```
- Only plain objects become params. A string as the last argument is the context. A plain object as the first argument is the message. https://docs.nestjs.com/application/logger#structured-logging-params
- Production: JSON on stdout, collection left to the platform: https://docs.nestjs.com/application/logger#logging-in-production
  ```typescript
  NestFactory.create(AppModule, {
    logger: new ConsoleLogger({ json: true, logLevels: ['log', 'warn', 'error', 'fatal'] }),
  });
  ```
- In JSON mode, params nest under `params` unless `flattenParams: true` (keys clashing with `message` or `level` are dropped). `timestamp` is epoch ms and `prefix` is ignored. Errors go to stderr. `colors: true` or `compact: false` break strict JSON. https://docs.nestjs.com/application/logger#json-logging

## Custom logger with DI and buffered logs

```typescript
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(MyLogger));
```
https://docs.nestjs.com/application/logger#dependency-injection

- At least one module must import the module providing `MyLogger`. A logger instance passed directly as `logger` cannot inject providers. https://docs.nestjs.com/application/logger#dependency-injection
- The buffer flushes at `app.listen()` (or `useLogger()` for an application context). If a factory or hook calls `process.exit()`, or a startup step never settles, buffered logs are never printed: throw instead. https://docs.nestjs.com/application/logger#dependency-injection
- A context-aware injectable logger must be `Scope.TRANSIENT`, otherwise `setContext()` changes the context app-wide. https://docs.nestjs.com/application/logger#injecting-a-custom-logger
- Extending `ConsoleLogger`: call `super` in overridden methods. https://docs.nestjs.com/application/logger#extend-built-in-logger
