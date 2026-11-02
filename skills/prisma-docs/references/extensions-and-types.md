# Client extensions and generated types

## Contents
- How extensions behave
- model component
- query component
- result component
- Shared extensions
- Generated types and type utilities
- TypeScript performance

## How extensions behave

- 4 components: `model`, `client`, `query`, `result`. `$extends` returns a new client and never mutates the original: use the returned constant everywhere. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/shared-extensions
- Extended clients share the connection pool of their base client, unless a third-party extension builds its own client. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions
- `name` is optional: when set, it appears in error logs. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions
- Chained extensions: on conflict the last one wins. Query extensions run in declaration order. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions
- `$on` and `$use` do not exist on an extended client: register `$on` before `$extends`. Check that `$transaction` or `$connect` exist before calling them on an extended client. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#client-methods
- Type of an extended client: `type X = typeof xprisma`, or `ReturnType<typeof getExtendedClient>` for a factory. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions

## model component

- Add methods to one model or to all with `$allModels`. Inside, call other methods through `Prisma.getExtensionContext(this)`, not through the outer `prisma` (not extended). `.$name` gives the model name at runtime. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/model
  ```ts
  const xprisma = prisma.$extends({
    model: {
      $allModels: {
        async exists<T>(this: T, where: Prisma.Args<T, "findFirst">["where"]): Promise<boolean> {
          const context = Prisma.getExtensionContext(this);
          return (await (context as any).findFirst({ where })) !== null;
        },
      },
    },
  });
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/model

## query component

- The callback receives `{ model, operation, args, query }` and returns `query(args)`. Levels: `query.user.findMany`, `query.$allModels.findMany`, `query.user.$allOperations`, `query.$allModels.$allOperations`, `query.$allOperations` (also raw queries, where `model` is `undefined`). https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/query
- You may mutate `args` (for example `args.where`), except `include` and `select` (the output type would change and break type safety). https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/query
- The `query` component does not support nested read and write operations. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions
- Running a query inside a batch transaction (row-level security pattern): https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/query
  ```ts
  Prisma.defineExtension((prisma) => prisma.$extends({
    query: { user: { async findFirst({ args, query }) {
      const [result] = await prisma.$transaction([query(args)]);
      return result;
    } } },
  }));
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/query
- To override existing fields (for example mask `password`), prefer `result` over `query` for performance: `result` computes only on access, `query` computes after query execution. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/query

## result component

- Computed fields are evaluated on access. `needs` references scalar fields of the same model only, never relations. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/result
- Computed fields cannot be aggregated, and a `select` that leaves them out returns `undefined` for them. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/result
- Omitting a dependency of a computed field still reads it from the database. Omit the computed field too. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/result

## Shared extensions

- Define with `Prisma.defineExtension({...})` for type checks, and use `$allModels`/`$allOperations` keys. Use the function form `(client) => client.$extends({...})` to call client-level methods. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/shared-extensions
- An extension that calls `client.$transaction` or `$queryRaw` while triggered inside a transaction runs on a new connection, outside that transaction. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/shared-extensions
- Publish as `prisma-extension-<name>`. The only first-party extension listed is `@prisma/extension-read-replicas`. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/shared-extensions https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/extension-examples

## Generated types and type utilities

- Import `Prisma` from the generated client path. https://www.prisma.io/docs/orm/v7/prisma-client/type-safety
- Default to the safe `Input` types (relations through `connect`) over `UncheckedInput` (raw `authorId`). https://www.prisma.io/docs/orm/v7/prisma-client/type-safety
- Derive partial model types instead of writing interfaces: https://www.prisma.io/docs/orm/v7/prisma-client/type-safety/operating-against-partial-structures-of-model-types
  ```ts
  const userWithPosts = { include: { posts: true } } satisfies Prisma.UserDefaultArgs;
  type UserWithPosts = Prisma.UserGetPayload<typeof userWithPosts>;
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/type-safety/operating-against-partial-structures-of-model-types
- Utilities: `Prisma.Args<typeof prisma.post, "create">["data"]`, `Prisma.Result<T, A, "findMany">`, `Prisma.Payload<T, Operation>`, `Prisma.Exact<Input, Shape>`. https://www.prisma.io/docs/orm/v7/prisma-client/client-extensions/type-utilities
- Always validate and sanitize user input (the page uses zod) before database operations. https://www.prisma.io/docs/orm/v7/more/best-practices

## TypeScript performance

- On large schemas, type parameters as `typeof client` instead of `PrismaClient`. The docs' benchmark drops compile time from 1.86 s to 0.41 s. https://www.prisma.io/docs/orm/v7/more/troubleshooting/typescript-performance
  ```ts
  const client = new PrismaClient({ adapter });
  const saveFn = async (prismaClient: typeof client) => { /* ... */ };
  ```
  https://www.prisma.io/docs/orm/v7/more/troubleshooting/typescript-performance
- With the `prisma-client` generator, import from `enums` and `models/<Name>` for finer-grained imports, which can improve compile and typecheck performance. The top-level barrel files still work. https://www.prisma.io/docs/orm/v7/prisma-schema/overview/generators
