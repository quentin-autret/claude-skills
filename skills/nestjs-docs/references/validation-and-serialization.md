# Validation and serialization

## Contents
- Choosing a pipe
- Standard Schema (Zod, Valibot, ArkType)
- class-validator and ValidationPipe
- Mapped types
- Serialization
- WebSockets and microservices

## Choosing a pipe

- Schema-first option: the built-in `StandardSchemaValidationPipe` and the `schema` option of `@Body()`, `@Query()`, `@Param()`, `@RawBody()`, `@MessageBody()`, `@Payload()`. The decorator-based `ValidationPipe` with class-validator DTOs remains fully supported, with no plans to remove it. https://docs.nestjs.com/migration-guide#route-decorator-schemas https://docs.nestjs.com/application/validation#schema-based-validation
- The `schema` option only attaches metadata. Nothing is validated until a pipe reads it: https://docs.nestjs.com/migration-guide#route-decorator-schemas
  ```typescript
  app.useGlobalPipes(new StandardSchemaValidationPipe());
  // ...
  create(@Body({ schema: createUserSchema }) body: CreateUserDto)
  ```
- Both pipes can be bound together: `ValidationPipe` only validates class-typed parameters, `StandardSchemaValidationPipe` only parameters that declare a schema, so schemas can be adopted incrementally. https://docs.nestjs.com/application/validation#overview
- A `schema` on its own only attaches metadata. Bind a validation pipe (globally, for example) so the rules run. https://docs.nestjs.com/migration-guide#route-decorator-schemas https://docs.nestjs.com/application/validation#auto-validation

## Standard Schema (Zod, Valibot, ArkType)

- The DTO type is the schema type: `export type CreateUserDto = z.infer<typeof createUserSchema>`. Single values: `@Body('email', { schema: z.email() })`. https://docs.nestjs.com/application/validation#schema-based-validation
- `transform` defaults to `true`: the handler receives the schema output (coercion, defaults, transforms applied). With `transform: false` type the parameter with `z.input<typeof schema>`. https://docs.nestjs.com/application/validation#schema-validation-options
- Path and query params are strings: use coercing schemas, `z.coerce.number().int().positive()`. https://docs.nestjs.com/application/validation#coercion-and-transformation
- Unknown keys are the schema's job, not the pipe's: `z.object()` strips, `z.strictObject()` rejects, `z.looseObject()` keeps. https://docs.nestjs.com/application/validation#unknown-properties
- Derive schemas with `.partial()`, `.pick()`, `.omit()`, `.extend()` instead of mapped types. https://docs.nestjs.com/application/validation#deriving-schemas
- Custom param decorators are skipped unless `validateCustomDecorators: true`. https://docs.nestjs.com/application/validation#validating-custom-decorators
- Errors are 400 with messages prefixed by the path (`email: Invalid email address`, `1.email: ...`). Change the shape with `exceptionFactory`. https://docs.nestjs.com/application/validation#schema-validation-options https://docs.nestjs.com/application/validation#deriving-schemas
- Zod requires `strictNullChecks`. https://docs.nestjs.com/pipes#binding-validation-pipes

## class-validator and ValidationPipe

```typescript
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```
https://docs.nestjs.com/application/validation#stripping-properties

- `whitelist` strips undecorated properties. `forbidNonWhitelisted` rejects them. `transform` builds DTO instances and converts primitive params by declared type. https://docs.nestjs.com/application/validation#transform-payload-objects
- No metadata exists for generics or interfaces: DTOs must be concrete classes, and `import type` erases them. https://docs.nestjs.com/application/validation#auto-validation
- `CreateUserDto[]` bodies are not validated element by element. Use `new ParseArrayPipe({ items: CreateUserDto })` or a wrapper class. https://docs.nestjs.com/application/validation#parsing-and-validating-arrays
- `errorFormat: 'grouped'` returns an object keyed by property path instead of the default string list. `stopAtFirstError` defaults to `false`. https://docs.nestjs.com/application/validation#using-the-built-in-validationpipe
- `disableErrorMessages: true` hides details. https://docs.nestjs.com/application/validation#disable-detailed-errors
- Cookie params have `type === 'custom'`, so a global `ValidationPipe` skips them without `validateCustomDecorators`. https://docs.nestjs.com/http/cookies#reading-cookies

## Mapped types

- `PartialType`, `PickType`, `OmitType`, `IntersectionType` compose: `PartialType(OmitType(CreateCatDto, ['name'] as const))`. https://docs.nestjs.com/application/validation#mapped-types
- Import them from `@nestjs/swagger` when using Swagger, from `@nestjs/graphql` when using GraphQL, otherwise from `@nestjs/mapped-types`. Importing from `@nestjs/mapped-types` instead of the Swagger or GraphQL packages can cause undocumented side effects. https://docs.nestjs.com/application/validation#mapped-types https://docs.nestjs.com/openapi/mapped-types

## Serialization

- Schema-based: `StandardSchemaSerializerInterceptor` with `@SerializeOptions({ schema })`. A response schema is an allowlist (`z.object()` drops `password`). https://docs.nestjs.com/application/serialization#schema-based-serialization
  ```typescript
  @UseInterceptors(StandardSchemaSerializerInterceptor)
  @SerializeOptions({ schema: userResponseSchema })
  ```
- For arrays pass the item schema, not an array schema. `null`, `undefined`, primitives and `StreamableFile` pass unchanged. https://docs.nestjs.com/application/serialization#schema-based-serialization
- A response that does not match its schema becomes a 500 (treated as a server bug), not partial data. https://docs.nestjs.com/application/serialization#serialization-errors
- Global binding: `app.useGlobalInterceptors(new StandardSchemaSerializerInterceptor(app.get(Reflector)))` or `APP_INTERCEPTOR`. Handler schema wins over the controller one. https://docs.nestjs.com/application/serialization#binding-the-schema-serializer
- class-transformer: `ClassSerializerInterceptor` serializes only class instances. A plain object, even `{ user: new UserEntity() }`, is not serialized correctly. https://docs.nestjs.com/application/serialization#exclude-properties
- With `@SerializeOptions({ type: UserEntity })`, the response is shaped by `type`: an instance of another entity loses its own `@Exclude()`, and undeclared properties are copied. Add `@Expose()` on every property and `excludeExtraneousValues: true`. https://docs.nestjs.com/application/serialization#transform-plain-objects
- Neither interceptor serializes `StreamableFile`. https://docs.nestjs.com/application/serialization
- Do not use `ClassSerializerInterceptor` with MikroORM entities (infinite recursion on collections). https://docs.nestjs.com/data/mikroorm#serialization

## WebSockets and microservices

- Both pipes throw an HTTP exception by default, which WebSocket and microservice clients receive as `Internal server error`. Return a `WsException` or an `RpcException` from `exceptionFactory`: https://docs.nestjs.com/application/validation#websockets-and-microservices https://docs.nestjs.com/websockets/pipes https://docs.nestjs.com/microservices/pipes
  ```typescript
  new StandardSchemaValidationPipe({
    exceptionFactory: (issues) => new WsException(issues.map((issue) => issue.message)),
  })
  ```
- Both serializer interceptors behave the same on WebSockets and microservices. https://docs.nestjs.com/application/serialization#websockets-and-microservices
