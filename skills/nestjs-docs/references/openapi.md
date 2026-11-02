# OpenAPI (Swagger): setup, types, operations, security, CLI plugin

## Contents
- Setup
- Standard Schema (Zod, Valibot)
- Types and parameters
- Operations and responses
- Security
- Mapped types
- CLI plugin
- Other features
- Old patterns

## Setup

- On Fastify also install `@fastify/static`. https://docs.nestjs.com/openapi/introduction#installation
- Pass a factory to `setup()` so the document is generated on first request, not at init: https://docs.nestjs.com/openapi/introduction#bootstrap
  ```typescript
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, documentFactory);
  ```
- If `createDocument()` is called eagerly, `app.enableVersioning()` must run before it, otherwise `paths` miss the version prefix. The factory is unaffected. https://docs.nestjs.com/openapi/introduction#bootstrap
- JSON is served at `<path>-json` and YAML at `<path>-yaml` (`/api-json` for UI at `/api`). Change with `jsonDocumentUrl` / `yamlDocumentUrl`. https://docs.nestjs.com/openapi/introduction#setup-options
- `useGlobalPrefix` defaults to `false`: the Swagger routes are not prefixed by `setGlobalPrefix()`. https://docs.nestjs.com/openapi/introduction#setup-options
- `ui` and `raw` are independent. `{ ui: false, raw: ['json'] }` leaves only `/api-json`. `raw: false` or `[]` serves no document. https://docs.nestjs.com/openapi/introduction#setup-options
- Swagger UI works with the default CSP of Nest's built-in security headers. Adjust directives only when adding `customJs`/`customJsStr` or serving over plain HTTP outside local development (https://docs.nestjs.com/security/helmet#swagger-ui-and-graphql-ides). https://docs.nestjs.com/openapi/introduction#bootstrap
- `helmet` registered directly on Fastify can block Swagger UI. Allow `'unsafe-inline'` styles and scripts plus `data:` and `validator.swagger.io` images (or set `contentSecurityPolicy: false`). https://docs.nestjs.com/openapi/introduction#bootstrap

### Document options (third argument of `createDocument`)

- Default `operationId` is `controllerKey_methodKey_version` (`UsersController_createUser`). `operationIdFactory: (controllerKey, methodKey) => methodKey` gives `createUser`. https://docs.nestjs.com/openapi/introduction#document-options
- `autoTagControllers` defaults to `true` (tag = controller name without `Controller`). With `false`, use `@ApiTags()`. https://docs.nestjs.com/openapi/introduction#document-options
- `excludeDynamicDefaults: true` drops `default` values that are non-plain objects (`new Date()`, class instances) so the document stays stable across restarts. https://docs.nestjs.com/openapi/introduction#document-options

## Standard Schema (Zod, Valibot)

- Param decorators take a Standard Schema, which Swagger turns into bodies and params: `@Body({ schema: createCatSchema }) dto: CreateCatDto`. https://docs.nestjs.com/openapi/introduction#standard-schema-zod-valibot
- Libraries with Standard JSON Schema (`~standard.jsonSchema`, Zod 4.2+) need no configuration. Nest uses the `openapi-3.0` target, `input` for requests and `output` for responses. https://docs.nestjs.com/openapi/introduction#libraries-that-need-no-configuration
- Otherwise set `standardSchemaConverter(schema, { schemaType })` returning `{ schema, components? }` (`undefined` falls back to native conversion). Pass `schemaType` through, never hardcode it. https://docs.nestjs.com/openapi/introduction#supplying-a-converter
  ```typescript
  import { createSchema } from 'zod-openapi';
  standardSchemaConverter: (schema, { schemaType }) =>
    schema['~standard']?.vendor === 'zod'
      ? createSchema(schema as never, { io: schemaType, openapiVersion: '3.0.0' })
      : undefined,
  ```
  https://docs.nestjs.com/openapi/introduction#supporting-several-libraries-at-once
- Always branch on `~standard.vendor`: a Valibot schema passed to `createSchema()` (or the reverse) throws at document generation. Valibot uses `toJsonSchema(schema as never, { target: 'openapi-3.0', typeMode: schemaType })` from `@valibot/to-json-schema`. https://docs.nestjs.com/openapi/introduction#supporting-several-libraries-at-once

## Types and parameters

- A DTO shows an empty model unless its properties have `@ApiProperty()` or the CLI plugin runs. https://docs.nestjs.com/openapi/types-and-parameters
- Without the plugin, declare arrays (`type: [String]`) and circular types (`type: () => Node`). https://docs.nestjs.com/openapi/types-and-parameters#arrays https://docs.nestjs.com/openapi/types-and-parameters#circular-dependencies
- Generics lose metadata: `@Body() dto: CreateUserDto[]` needs `@ApiBody({ type: [CreateUserDto] })`. https://docs.nestjs.com/openapi/types-and-parameters#generics-and-interfaces
- `enum` is inlined in each parameter, so client generators duplicate it. Add `enumName` (also on `@ApiQuery()`, `@ApiParam()`) for a shared component: https://docs.nestjs.com/openapi/types-and-parameters#enums-schema
  ```typescript
  @ApiProperty({ enum: CatBreed, enumName: 'CatBreed' })
  ```
- Models referenced only by `getSchemaPath()` (`oneOf`, `anyOf`, `allOf`) must be registered with `@ApiExtraModels()` or `extraModels` in document options. Polymorphic arrays need `type: 'array', items: { oneOf: [...] }`. https://docs.nestjs.com/openapi/types-and-parameters#oneof-anyof-allof

## Operations and responses

- OpenAPI 3.2 tag hierarchy (`parent`, `kind`) needs `setOpenAPIVersion('3.2.0')` and only works through `DocumentBuilder.addTag(name, description, undefined, { parent })`. `@ApiTags()` ignores them with a warning. https://docs.nestjs.com/openapi/operations#tags
- `@ApiExtension()` names must start with `x-`, else it throws. https://docs.nestjs.com/openapi/operations#extensions
- Generic paginated response: leave `results` undecorated, register the models with `ApiExtraModels` and add a `title` so generated clients get a named type: https://docs.nestjs.com/openapi/operations#advanced-generic-apiresponse
  ```typescript
  applyDecorators(ApiExtraModels(PaginatedDto, model), ApiOkResponse({ schema: {
    title: `PaginatedResponseOf${model.name}`,
    allOf: [{ $ref: getSchemaPath(PaginatedDto) },
      { properties: { results: { type: 'array', items: { $ref: getSchemaPath(model) } } } }],
  } }));
  ```

## Security

- Each decorator pairs with a `DocumentBuilder` definition: `@ApiBearerAuth()` + `addBearerAuth()`, `@ApiBasicAuth()` + `addBasicAuth()`, `@ApiOAuth2(['pets:write'])` + `addOAuth2()`, `@ApiCookieAuth()` + `addCookieAuth('optional-session-id')`, `@ApiSecurity(name)` + `addSecurity(name, {...})`. https://docs.nestjs.com/openapi/security

## Mapped types

- Import `PartialType`, `PickType`, `OmitType`, `IntersectionType` from `@nestjs/swagger`, not `@nestjs/mapped-types`, so the schema (and the plugin) sees them. Keys use `as const`. https://docs.nestjs.com/openapi/mapped-types https://docs.nestjs.com/openapi/cli-plugin

## CLI plugin

- Enable with `"compilerOptions": { "plugins": ["@nestjs/swagger"] }` in `nest-cli.json`. It adds `@ApiProperty()`, `required` from `?`, types, defaults and a response decorator per endpoint. An explicit `@ApiProperty()` overrides. It only documents: validation still needs `class-validator`. https://docs.nestjs.com/openapi/cli-plugin#overview
- Only files ending in `.dto.ts` / `.entity.ts` (and `.controller.ts`) are analysed. Change with `dtoFileNameSuffix`. https://docs.nestjs.com/openapi/cli-plugin#overview
- Surprising defaults: `classValidatorShim true`, `introspectComments false`, `skipAutoHttpCode false` (the plugin adds `@HttpCode()`). `classTransformerShim: true` skips `@Exclude()` fields. https://docs.nestjs.com/openapi/cli-plugin#using-the-cli-plugin
- With `introspectComments`, the first comment line becomes `summary`. `@remarks`, `@deprecated`, `@example`, `@throws {400}` and `@param` are read. `@param` matches the variable name (`@Query('order_by') orderBy` is `@param orderBy`). Explicit `@ApiQuery()`/`@ApiParam()` win. https://docs.nestjs.com/openapi/cli-plugin#comments-introspection
- Delete `dist/` and rebuild after changing plugin options. https://docs.nestjs.com/openapi/cli-plugin#using-the-cli-plugin
- SWC: run `nest start -b swc --type-check`. In a monorepo, generate metadata and call `await SwaggerModule.loadPluginMetadata(metadata)` before `createDocument`. https://docs.nestjs.com/openapi/cli-plugin#swc-builder
- ts-jest does not apply the plugin. Add an AST transformer exporting `name`, `version` (bump it on every change so Jest notices) and `factory = (cs) => require('@nestjs/swagger/plugin').before({}, cs.program)`. Clear stale builds with `npx jest --clearCache`. https://docs.nestjs.com/openapi/cli-plugin#integration-with-ts-jest-e2e-tests https://docs.nestjs.com/openapi/cli-plugin#troubleshooting-jest-e2e-tests

## Other features

- `addGlobalParameters({ name: 'tenantId', in: 'header' })` and `addGlobalResponse({ status: 500, description })` on `DocumentBuilder`. https://docs.nestjs.com/openapi/other-features#global-parameters
- Several specs: `createDocument(app, config, { include: [CatsModule] })` with one `setup()` per spec. For a dropdown, set `explorer: true`, `swaggerOptions.urls: [{ name, url }]` and a `jsonDocumentUrl` per spec. https://docs.nestjs.com/openapi/other-features#multiple-specifications https://docs.nestjs.com/openapi/other-features#dropdown-in-the-explorer-bar

## Old patterns

| Old | Current |
| --- | --- |
| `swaggerUiEnabled` | `ui` |
| `validatorUrl`, `url`, `urls` setup options | none (no effect) |
| Mapped types from `@nestjs/mapped-types` | from `@nestjs/swagger` |
| Jest < 29 `globals.ts-jest.astTransformers` | `transform: { "^.+\\.(t|j)s$": ["ts-jest", { astTransformers: { before: [path] } }] }` |

https://docs.nestjs.com/openapi/introduction#setup-options https://docs.nestjs.com/openapi/mapped-types https://docs.nestjs.com/openapi/cli-plugin#integration-with-ts-jest-e2e-tests
