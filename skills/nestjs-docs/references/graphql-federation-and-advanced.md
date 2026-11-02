# GraphQL: federation, mapped types, middleware, plugins, SDL

## Mapped types

- Import `PartialType`, `PickType`, `OmitType`, `IntersectionType` from `@nestjs/graphql` (code first only). Key lists take `as const`, and types compose: https://docs.nestjs.com/graphql/mapped-types#composition
  ```typescript
  export class UpdateUserInput extends PartialType(OmitType(CreateUserInput, ['email'] as const)) {}
  ```
- Parent with another decorator: `PartialType(User, InputType)`. Options object: `decorator`, `omitDefaultValues`, `skipNullProperties` (default `true`, set `false` to still validate `null`). https://docs.nestjs.com/graphql/mapped-types#partial
- Metadata from `getFieldsAndDecoratorForType()` is shared with the schema builder: treat it as read-only and pass copies (`{ ...field.options }`) to `@Field()`. A class without a GraphQL type decorator throws `UnableToFindFieldsError`. https://docs.nestjs.com/graphql/mapped-types#building-your-own

## CLI plugin

- Opt-in, code first only. Adds `@Field()` to input, object and args classes (except `@HideField`), `nullable` from `?`, the type from the TypeScript type (arrays included), only in files ending `.input.ts`, `.args.ts`, `.entity.ts` or `.model.ts` (`typeFileNameSuffix`). An explicit `@Field()` wins. https://docs.nestjs.com/graphql/cli-plugin#overview
  ```json
  { "compilerOptions": { "plugins": [{ "name": "@nestjs/graphql", "options": { "introspectComments": true } }] } }
  ```
  https://docs.nestjs.com/graphql/cli-plugin#using-the-cli-plugin
- Custom webpack with `ts-loader`: `before: [require('@nestjs/graphql/plugin').before({}, program)]`. https://docs.nestjs.com/graphql/cli-plugin#using-the-cli-plugin
- SWC needs `nest start -b swc --type-check`. In a monorepo run `generate-metadata.ts` and pass `metadata` from `./metadata.js` to `GraphQLModule.forRoot`. https://docs.nestjs.com/graphql/cli-plugin#swc-builder
- `Object type <name> must define one or more fields.` in `ts-jest` e2e means jest never loaded the plugin: add a transformer (`name`, `version`, `factory` calling `transformer.before({}, cs.program)`) as `astTransformers.before` under `transform` for `ts-jest`. Bump `version` on every config change or jest keeps its cache. https://docs.nestjs.com/graphql/cli-plugin#integration-with-ts-jest-e2e-tests

## Directives

- Custom directive: a transformer built with `mapSchema`, `getDirective`, `MapperKind` from `@graphql-tools/utils` wrapping `fieldConfig.resolve`, registered with `transformSchema: (schema) => upperDirectiveTransformer(schema, 'upper')`. https://docs.nestjs.com/graphql/directives#custom-directives
- Code first: `@Directive('@upper')`, declared in `buildSchemaOptions: { directives: [new GraphQLDirective({ name: 'upper', locations: [DirectiveLocation.FIELD_DEFINITION] })] }`. Directives applied with `@Directive()` do not appear in the generated schema file. https://docs.nestjs.com/graphql/directives#code-first
- Schema first: `directive @upper on FIELD_DEFINITION` in SDL. https://docs.nestjs.com/graphql/directives#schema-first

## Field middleware and extensions

- Code first only, on `ObjectType` classes only. A returned `undefined` keeps the value of `next()`: https://docs.nestjs.com/graphql/field-middleware#getting-started
  ```typescript
  const loggerMiddleware: FieldMiddleware = async (ctx: MiddlewareContext, next: NextFn) => {
    const value = await next();
    return value;
  };
  ```
- Field middleware has no DI access and must stay light (no database reads). For services, use a guard or interceptor on the root query or mutation and put the result on `context`. https://docs.nestjs.com/graphql/field-middleware#getting-started
- Bind with `@Field({ middleware: [...] })`, `@ResolveField(() => String, { middleware })` or `@UseFieldMiddleware()`. Order: global `buildSchemaOptions.fieldMiddleware`, then the `middleware` option, then `@UseFieldMiddleware()`. The first in an array is outermost. With field-level enhancers, middleware runs after root enhancers but before method guards and interceptors. https://docs.nestjs.com/graphql/field-middleware#global-field-middleware
- Field permissions: `@Extensions({ role: Role.ADMIN })`, read in middleware through `info.parentType.getFields()[info.fieldName].extensions`, compared to the real user from `ctx.context`, then throw `ForbiddenException` or return `null`. https://docs.nestjs.com/graphql/extensions#using-custom-metadata

## Plugins and complexity

- Apollo plugin: a class with `@Plugin()` from `@nestjs/apollo` implementing `ApolloServerPlugin`, registered as a provider (applied automatically). External plugins go in `plugins: [...]`. https://docs.nestjs.com/graphql/plugins#custom-plugins https://docs.nestjs.com/graphql/plugins#using-external-plugins
- Mercurius: Fastify plugins that must load after Mercurius go in `plugins: [{ plugin, options }]`, except `mercurius-upload`, registered in the main file. https://docs.nestjs.com/graphql/plugins#plugins-with-mercurius
- Query complexity (code first, `graphql-query-complexity`): an `@Plugin()` that injects `GraphQLSchemaHost` and, in `didResolveOperation`, throws a `GraphQLError` above a maximum. The first estimator returning a number decides: https://docs.nestjs.com/graphql/complexity#getting-started
  ```typescript
  const complexity = getComplexity({ schema, operationName: request.operationName, query: document,
    variables: request.variables, estimators: [fieldExtensionsEstimator(), simpleEstimator({ defaultComplexity: 1 })] });
  ```
- Per field: `@Field({ complexity: 3 })`. On queries: `complexity: (options: ComplexityEstimatorArgs) => options.args.count * options.childComplexity`. https://docs.nestjs.com/graphql/complexity#field-level-complexity

## Generating SDL and sharing models

- Without starting the server: `NestFactory.create(GraphQLSchemaBuilderModule)`, `app.init()`, `app.get(GraphQLSchemaFactory).create([RecipesResolver])`, then `printSchema()` from `graphql`. Unreferenced classes are omitted unless listed in `orphanedTypes`. https://docs.nestjs.com/graphql/generating-sdl#usage
- `GraphQLSchemaFactory` ignores Federation: print with `printSubgraphSchema` from `@apollo/subgraph`. For Federation 2, prepend `extend schema @link(url: "https://specs.apollo.dev/federation/v2.14", import: [...])` listing every directive used, or the schema is read as Federation 1. Put `_entities`-only stubs in `orphanedTypes`. https://docs.nestjs.com/graphql/generating-sdl#federation
- Frontend sharing models: bundlers honoring the `browser` export condition get a decorator shim. Otherwise alias `@nestjs/graphql` to `node_modules/@nestjs/graphql/dist/extra/graphql-model-shim`. https://docs.nestjs.com/graphql/sharing-models#using-the-model-shim

## Custom driver

- Extend `AbstractGraphQLDriver` with `start(options)` and `stop()`. `options.schema` is already built when `start()` runs, and `this.httpAdapterHost` gives the adapter. https://docs.nestjs.com/graphql/other-features#creating-a-custom-driver

## Federation

- Apollo subgraph: `npm install --save @apollo/subgraph`, `driver: ApolloFederationDriver` with `typePaths` or `autoSchemaFile`. Gateway: `@apollo/gateway`. https://docs.nestjs.com/graphql/federation#federation-with-apollo https://docs.nestjs.com/graphql/federation#schema-first
- Entity owner: `@Directive('@key(fields: "id")')` plus a `@ResolveReference()` method receiving `{ __typename, id }`. https://docs.nestjs.com/graphql/federation#code-first
- Referencing another subgraph: return `{ __typename: 'User', id: post.authorId }` from the field resolver, and list the foreign type in `buildSchemaOptions: { orphanedTypes: [User] }`. https://docs.nestjs.com/graphql/federation#federated-example-posts https://docs.nestjs.com/graphql/federation#code-first-1
- Federation 2, code first: `autoSchemaFile: { federation: 2 }` (Nest adds `@link`). Drop `extend`, `@extends` and `@external`. Schema first: add `extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key"])`. https://docs.nestjs.com/graphql/federation#federation-2
- Gateway: https://docs.nestjs.com/graphql/federation#federated-example-gateway
  ```typescript
  GraphQLModule.forRoot<ApolloGatewayDriverConfig>({ driver: ApolloGatewayDriver,
    gateway: { supergraphSdl: new IntrospectAndCompose({ subgraphs: [{ name: 'users', url: usersUrl }] }) } })
  ```
- `ApolloGatewayDriver` does not forward subscriptions: use a router supporting federated subscriptions (Apollo Router) or the Mercurius gateway (`subscription: true`, `wsUrl` per service). https://docs.nestjs.com/graphql/federation
- Mercurius: `MercuriusFederationDriver` (still needs `@apollo/subgraph`) and `MercuriusGatewayDriver` from `@mercuriusjs/gateway` with `gateway: { services: [{ name, url }] }`. Mercurius does not fully support Federation 2. https://docs.nestjs.com/graphql/federation#federation-with-mercurius https://docs.nestjs.com/graphql/federation#federation-2
- `@Directive('@inaccessible')` hides an operation from the supergraph, while it stays executable on the subgraph. https://docs.nestjs.com/graphql/federation#hiding-operations-from-the-supergraph
