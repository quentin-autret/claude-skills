# Data: TypeORM, Sequelize, Mongoose, MikroORM

## Contents
- TypeORM
- Sequelize
- Mongoose
- MikroORM v7

## TypeORM

- Never `synchronize: true` in production: data can be lost. https://docs.nestjs.com/data/typeorm
- Prefer `autoLoadEntities: true` over a root `entities` array. Entities only reachable through a relation and never passed to `forFeature()` are not loaded. https://docs.nestjs.com/data/typeorm#auto-load-entities
- Defaults: `retryAttempts` 10, `retryDelay` 3000 ms, every error retried. `manualInitialization: true` skips connecting and migrations at init. https://docs.nestjs.com/data/typeorm
- To use a repository elsewhere, export `TypeOrmModule` from the module calling `forFeature()`. `DataSource` and `EntityManager` are injectable everywhere. https://docs.nestjs.com/data/typeorm#repository-pattern
- Transactions: the docs recommend `QueryRunner` (release it in `finally`), the alternative is `dataSource.transaction(async (manager) => ...)`. Wrap `DataSource` behind a small factory interface to test it. https://docs.nestjs.com/data/typeorm#typeorm-transactions
- Named data sources need `name` set next to `useFactory` in `forRootAsync()`, and the name passed to `forFeature()`, `@InjectRepository()`, `@InjectDataSource()`. https://docs.nestjs.com/data/typeorm#multiple-databases
- Subscribers cannot be request-scoped. With `EntitySchema`, `name` must match the `target` class. https://docs.nestjs.com/data/typeorm#subscribers
- Store entities loaded by the CLI need explicit column `type`s (no decorator metadata there). Never `save()` for conditional writes (it reads the row first): use `update()`/`delete()` on the entity manager with the conditions in the criteria, and read `affected`. https://docs.nestjs.com/reliability/idempotency#store-keys-in-postgresql
- SWC breaks circular relation imports: type them `Relation<Profile>`. https://docs.nestjs.com/recipes/swc#common-pitfalls

## Sequelize

- Defaults: `synchronize: true` (for auto-loaded models), `autoLoadModels: false`, `retryAttempts` 10. Models reachable only through associations are not auto-loaded. https://docs.nestjs.com/data/sequelize#auto-load-models
- Inject with `@InjectModel(User) private userModel: typeof User`. Export `SequelizeModule` to share models. https://docs.nestjs.com/data/sequelize#models
- Managed transactions: `this.sequelize.transaction(async (t) => this.userModel.create(data, { transaction: t }))`. https://docs.nestjs.com/data/sequelize#sequelize-transactions

## Mongoose

- Collection name defaults to the class name with an `s` added at the end (`Cat` becomes `cats`). https://docs.nestjs.com/data/mongodb#model-injection
- Arrays and nested objects need explicit types: `@Prop([String]) tags: string[]`. References need `type` and `ref`: `@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })`. https://docs.nestjs.com/data/mongodb#model-injection
- Hooks (`pre()`/`post()`) added after the model compiles do nothing: register them in `forFeatureAsync()`. Global plugins go through `connectionFactory` (https://docs.nestjs.com/data/mongodb#plugins). https://docs.nestjs.com/data/mongodb#hooks-middleware
- Start sessions from `@InjectConnection()` (`connection.startSession()`), not `mongoose.startSession()`, and end them yourself. https://docs.nestjs.com/data/mongodb#sessions
- Several connections require `connectionName`, passed to `forFeature()`, `@InjectModel(Cat.name, 'cats')` and `@InjectConnection('cats')`. https://docs.nestjs.com/data/mongodb#multiple-databases
- `@Virtual()` defines a computed, never-persisted property. Subdocument typing uses `HydratedDocument<Person, Override>` (https://docs.nestjs.com/data/mongodb#subdocuments). https://docs.nestjs.com/data/mongodb#virtuals
- Tests: `{ provide: getModelToken(Cat.name), useValue: catModel }`. https://docs.nestjs.com/data/mongodb#testing

## MikroORM v7

- `@mikro-orm/nestjs` is a third-party package, not managed by the NestJS core team. The chapter targets v7. https://docs.nestjs.com/data/mikroorm
- v7 changes: decorators come from `@mikro-orm/decorators/legacy`, `EntityManager`/`EntityRepository`/`MikroORM` from the driver package, `metadataProvider: ReflectMetadataProvider` must be set explicitly, and `forRoot()` needs a config argument. https://docs.nestjs.com/data/mikroorm#installation
  ```typescript
  MikroOrmModule.forRoot({
    driver: SqliteDriver, dbName: 'my-db-name.sqlite3',
    entities: ['./dist/**/*.entity.js'], entitiesTs: ['./src/**/*.entity.ts'],
    metadataProvider: ReflectMetadataProvider,
  })
  ```
- In `forRootAsync()`, `driver` (and `contextName` for several databases) go next to `useFactory`, otherwise injecting the driver `EntityManager` fails. https://docs.nestjs.com/data/mikroorm#async-configuration https://docs.nestjs.com/data/mikroorm#multiple-databases
- Writes go through the `EntityManager` (Unit of Work): `em.flush()` writes pending changes in one transaction. https://docs.nestjs.com/data/mikroorm#repositories
- Outside a request context the global `EntityManager` throws on `find()` or `flush()`. In cron jobs and queue consumers put `@CreateRequestContext()` directly above the method, below `@Cron()`, or the job never registers. It needs an `orm` or `em` property on the class. https://docs.nestjs.com/data/mikroorm#request-scoped-handlers-in-queues
  ```typescript
  @Cron(CronExpression.EVERY_HOUR)
  @CreateRequestContext()
  async logPhotoCount() { await this.orm.em.count(Photo); }
  ```
- `scope: Scope.REQUEST` makes every provider using `EntityManager` request-scoped. https://docs.nestjs.com/data/mikroorm#request-context
- Never use `ClassSerializerInterceptor` with MikroORM entities (`Collection` recursion). Use MikroORM's `hidden`/`serialize()`. Wrap relation types in `Rel<T>` to avoid ESM circular-import errors. https://docs.nestjs.com/data/mikroorm#serialization
- Nested `em.transactional()` creates a savepoint, `@Transactional()` joins the current transaction. https://docs.nestjs.com/data/mikroorm#transactions
- `autoLoadEntities` does not affect the MikroORM CLI, which still needs the full list. https://docs.nestjs.com/data/mikroorm#load-entities-automatically
- Several databases: unique `contextName`, `registerRequestContext: false` on each, and `MikroOrmModule.forMiddleware()` once. https://docs.nestjs.com/data/mikroorm#multiple-databases

## Old patterns

| Old | Current |
| --- | --- |
| `@UseRequestContext()` (before MikroORM v6), then `@CreateRequestContext()` from `@mikro-orm/core` (before v7) | `@CreateRequestContext()` from `@mikro-orm/decorators/legacy` |
| decorators from `@mikro-orm/core` | `@mikro-orm/decorators/legacy` |
| `MikroOrmModule.forRoot()` with no argument | pass the config explicitly |

https://docs.nestjs.com/data/mikroorm#installation https://docs.nestjs.com/data/mikroorm#request-scoped-handlers-in-queues
