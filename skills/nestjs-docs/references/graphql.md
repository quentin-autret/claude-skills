# GraphQL: setup, resolvers, subscriptions, types

## Setup and IDE

- Express and Apollo: `npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/express5 graphql` (Fastify: `@as-integrations/fastify`). `@nestjs/apollo` v14 requires Apollo Server v5. https://docs.nestjs.com/graphql/quick-start#installation
- `GraphQLModule.forRoot<ApolloDriverConfig>({ driver: ApolloDriver })`. Options are forwarded to the `ApolloServer` constructor. https://docs.nestjs.com/graphql/quick-start#getting-started-with-graphql--typescript
- GraphiQL is the only built-in IDE. With Apollo it is on whenever `NODE_ENV` is not `production`, so set `graphiql: false` on publicly reachable non-production environments. `graphiql: true` forces it on in production too. https://docs.nestjs.com/graphql/quick-start#enabling-and-disabling-graphiql
- Apollo Sandbox instead: `graphiql: false` plus `plugins: [ApolloServerPluginLandingPageLocalDefault()]` from `@apollo/server/plugin/landingPage/default`. https://docs.nestjs.com/graphql/quick-start#apollo-sandbox
- Guards and interceptors reach the per-request context with `GqlExecutionContext.create(context).getContext()`. https://docs.nestjs.com/graphql/quick-start#accessing-the-request-and-response-objects
- `app.get(GraphQLSchemaHost).schema` only after `app.init()` or `app.listen()`. https://docs.nestjs.com/graphql/quick-start#accessing-generated-schema
- Multiple endpoints: `include: [CatsModule]` scopes only the resolver scan. Code-first types still land in every schema that references them unless they set `registerIn: () => CatsModule` (on `@ObjectType()`, `@InputType()`, `@InterfaceType()`, `@ArgsType()`, `registerEnumType()`, `createUnionType()`). Prefer the factory form to avoid circular imports. https://docs.nestjs.com/graphql/quick-start#multiple-endpoints

## Code first and schema first

- Code first: `autoSchemaFile: join(process.cwd(), 'src/schema.gql')` (or `true` for in memory). Types appear in module order, `sortSchema: true` sorts them. https://docs.nestjs.com/graphql/quick-start#code-first
- Schema first: `typePaths: ['./**/*.graphql']` plus `definitions: { path: join(process.cwd(), 'src/graphql.ts'), outputAs: 'class' }` (default output is interfaces). On demand: `new GraphQLDefinitionsFactory().generate({ typePaths, path, outputAs: 'class', watch: true })`. https://docs.nestjs.com/graphql/quick-start#schema-first
- Never add decorators to the generated file, they are overwritten: extend the generated class in another file. Classes allow `class-validator`, and validation needs `ValidationPipe`. https://docs.nestjs.com/graphql/resolvers#generating-types

## Resolvers and arguments

- Every field is non-nullable by default, opt out with `nullable: true`. Arrays: `nullable: 'items'` or `'itemsAndList'`. https://docs.nestjs.com/graphql/resolvers#object-types
- `number` maps to `Float` unless typed: `@Field(type => Int)`, `@Args('id', { type: () => Int }) id: number`. Arrays always need the type function. https://docs.nestjs.com/graphql/resolvers#object-types
- A class with `@ResolveField()` needs the parent type: `@Resolver(() => Author)` (schema first: `@Resolver('Author')`). Method-level `@Resolver()` is not supported in code first. https://docs.nestjs.com/graphql/resolvers#code-first-resolver https://docs.nestjs.com/graphql/resolvers#schema-first-resolver
- A nullable arg can deliver `null` or `undefined` even if the TypeScript type says otherwise. https://docs.nestjs.com/graphql/resolvers#args-decorator-options
- Prefer an `@ArgsType()` class over many `@Args()`: `@Args() args: GetAuthorArgs`, validated with `class-validator`. https://docs.nestjs.com/graphql/resolvers#dedicated-arguments-class
- Generic base resolver: `@Resolver({ isAbstract: true })` with an explicit return type. Generic types: a `Paginated(classRef)` factory returning `@ObjectType({ isAbstract: true })`, then `class PaginatedAuthor extends Paginated(Author) {}`. https://docs.nestjs.com/graphql/resolvers#class-inheritance https://docs.nestjs.com/graphql/resolvers#generics
- Resolvers are providers: list them in a module's `providers`. https://docs.nestjs.com/graphql/resolvers#module

## Mutations

- Object arguments use `@InputType()`. Expose the root query in a payload so clients refetch in one round trip: https://docs.nestjs.com/graphql/mutations#mutation-payload-query
  ```typescript
  @Field(() => 'Query') query: object;
  // return { todo, query: {} };
  ```

## Subscriptions

- Apollo: `subscriptions: { 'graphql-ws': true }`. Install `graphql-ws` only if you import from it. https://docs.nestjs.com/graphql/subscriptions#enable-subscriptions-with-apollo-driver
  ```typescript
  @Subscription(() => Comment, { filter: (payload, variables) => payload.commentAdded.title === variables.title })
  commentAdded() { return this.pubSub.asyncIterableIterator('commentAdded'); }
  ```
  https://docs.nestjs.com/graphql/subscriptions#filtering-subscriptions
- `publish('commentAdded', { commentAdded: newComment })`: the top-level key must be the subscription name. With a `resolve` option, return the unwrapped payload. Type `this` (`resolve(this: AuthorResolver, value)`) to use injected providers. https://docs.nestjs.com/graphql/subscriptions#publishing https://docs.nestjs.com/graphql/subscriptions#mutating-subscription-payloads
- The default `PubSub` of `graphql-subscriptions` is not for production: use one backed by an external store, provided once (`{ provide: 'PUB_SUB', useValue: new PubSub() }`) and injected. https://docs.nestjs.com/graphql/subscriptions#pubsub
- Authentication: `onConnect(context)` reads `connectionParams` and stores values in `context.extra`, then `context: ({ extra }) => ...`. https://docs.nestjs.com/graphql/subscriptions#authentication-over-websockets
- A custom `ws` server (`server: new WebSocketServer({ noServer: true, path, maxPayload })`) must use `noServer: true` (Nest throws otherwise), its `path` wins, one server per `GraphQLModule`. Only `ws` is supported. https://docs.nestjs.com/graphql/subscriptions#customize-subscriptions-server
- Mercurius: `subscription: true` (singular), `@Context('pubsub') pubSub`, `pubSub.publish({ topic, payload })`. Use `mqemitter-redis` as `emitter` in production, `verifyClient` for authentication. https://docs.nestjs.com/graphql/subscriptions#pubsub-1 https://docs.nestjs.com/graphql/subscriptions#authentication-over-websockets-1

## Scalars

- `Date` maps to `GraphQLISODateTime` and `number` to `Float`. Change with `buildSchemaOptions: { dateScalarMode: 'timestamp', numberScalarMode: 'integer' }`. https://docs.nestjs.com/graphql/scalars#code-first
- Override a scalar: `@Scalar('Date', () => Date) class DateScalar implements CustomScalar<number, Date>`, registered as a provider. Third-party or own `GraphQLScalarType`: `resolvers: { JSON: GraphQLJSON }` plus `@Field(() => GraphQLJSON)` (suite: `graphql-scalars`). https://docs.nestjs.com/graphql/scalars#override-a-default-scalar https://docs.nestjs.com/graphql/scalars#import-a-custom-scalar
- Generated schema-first types make custom scalars `any`: pass `defaultScalarType: 'unknown'` and `customScalarTypeMapping: { DateTime: 'Date' }` to `generate()`. https://docs.nestjs.com/graphql/scalars#schema-first

## Interfaces, unions and enums

- Interfaces are abstract classes with `@InterfaceType()`, never TypeScript interfaces. Implement with `@ObjectType({ implements: () => [Character] })`. https://docs.nestjs.com/graphql/interfaces#code-first
- Default `resolveType()` for interfaces and unions needs class instances, not object literals. Otherwise pass `resolveType(value)`. https://docs.nestjs.com/graphql/interfaces#code-first https://docs.nestjs.com/graphql/unions-and-enums#code-first
- An interface resolver (`@Resolver(() => Character)`) needs `inheritResolversFromInterfaces: true`. https://docs.nestjs.com/graphql/interfaces#interface-resolvers
- Unions: `createUnionType({ name: 'ResultUnion', types: () => [Author, Book] as const })`. Keep `as const` or the declaration file breaks for consumers. Return type `Array<typeof ResultUnion>`. https://docs.nestjs.com/graphql/unions-and-enums#code-first
- Schema first: a `@ResolveField() __resolveType(value)` returning the type name, on a resolver registered as a provider. https://docs.nestjs.com/graphql/unions-and-enums#schema-first
- Enums: `registerEnumType(AllowedColor, { name: 'AllowedColor', valuesMap: { BLUE: { deprecationReason: 'Too blue.' } } })`. Schema first internal values: `resolvers: { AllowedColor: { RED: '#f00' } }`. https://docs.nestjs.com/graphql/unions-and-enums#code-first-1 https://docs.nestjs.com/graphql/unions-and-enums#schema-first-1

## Enhancers

- Use `GqlExecutionContext.create(context)` in guards and interceptors, `GqlArgumentsHost.create(host)` in a `GqlExceptionFilter`. https://docs.nestjs.com/graphql/other-features#execution-context https://docs.nestjs.com/graphql/other-features#exception-filters
- Enhancers run only on root `@Query()`/`@Mutation()` methods, not field resolvers. Opt in with `fieldResolverEnhancers: ['interceptors', 'guards', 'filters']`, and skip unneeded work when `info.parentType.name` is not `Query` or `Mutation`, since field resolvers can run thousands of times. https://docs.nestjs.com/graphql/other-features#execute-enhancers-at-the-field-resolver-level

## Old patterns

| Old | Current |
| --- | --- |
| `playground: true`, graphql-playground IDE | `graphiql` (`playground` is a deprecated alias, `graphiql` wins) |
| `subscriptions: { 'subscriptions-transport-ws': ... }` | `'graphql-ws'` (wire-incompatible, update clients, `onConnect` receives the context) |
| Older Apollo Server | Apollo Server v5 with `@as-integrations/express5` |

https://docs.nestjs.com/graphql/quick-start#enabling-and-disabling-graphiql https://docs.nestjs.com/graphql/subscriptions#enable-subscriptions-with-apollo-driver https://docs.nestjs.com/graphql/quick-start#installation
