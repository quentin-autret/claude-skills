# Testing

## Unit tests with a mocked client

- `jest-mock-extended` gives a typed deep mock: `mockDeep<PrismaClient>()`. https://www.prisma.io/docs/orm/v7/prisma-client/testing/unit-testing
- Singleton pattern: mock the module that exports the client, reset the mock before each test, register the file in `setupFilesAfterEnv`. https://www.prisma.io/docs/orm/v7/prisma-client/testing/unit-testing
  ```ts
  jest.mock("./client", () => ({ __esModule: true, default: mockDeep<PrismaClient>() }));
  beforeEach(() => mockReset(prismaMock));
  export const prismaMock = prisma as unknown as DeepMockProxy<PrismaClient>;
  prismaMock.user.create.mockResolvedValue(user);
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/testing/unit-testing
- Dependency injection pattern: a `Context` holding the client and a `createMockContext()` per test. https://www.prisma.io/docs/orm/v7/prisma-client/testing/unit-testing
- Circular dependency errors while mocking: enable `"strictNullChecks": true`. https://www.prisma.io/docs/orm/v7/prisma-client/testing/unit-testing
- The docs' sample mixes a named export in `client.ts` (`export { prisma }`) with a default import in `singleton.ts` (`import prisma from "./client"`), so the two must be made consistent. https://www.prisma.io/docs/orm/v7/prisma-client/testing/unit-testing

## Integration tests on a real database

- Use a dedicated, disposable test database (a container), never dev or production. https://www.prisma.io/docs/orm/v7/prisma-client/testing/integration-testing
- The docs' test script starts the container, applies the committed migrations with `prisma migrate deploy`, then runs Jest with `-i`: `yarn docker:up && yarn prisma migrate deploy && jest -i` (`docker:up` is `docker compose up -d`). https://www.prisma.io/docs/orm/v7/prisma-client/testing/integration-testing
- Make sure the database is ready before the suite starts (the docs warn about this and their script has no wait). The Docker guide shows a compose `healthcheck` using `pg_isready` together with `depends_on: condition: service_healthy`. https://www.prisma.io/docs/orm/v7/prisma-client/testing/integration-testing https://www.prisma.io/docs/guides/v7/deployment/docker
- Keep the test `DATABASE_URL` in its own env file (`.env.test`), as part of a multiple `.env` files setup. https://www.prisma.io/docs/orm/v7/prisma-client/testing/integration-testing
- Seed in `beforeAll`, then clean in `afterAll` with ordered `deleteMany` calls inside `$transaction([...])` and `await prisma.$disconnect()`. https://www.prisma.io/docs/orm/v7/prisma-client/testing/integration-testing
- Wiping many tables: list the tables with `$queryRaw`, then `TRUNCATE ... CASCADE` with `$executeRawUnsafe`. It cannot be undone. The docs' code leaves out `_prisma_migrations`. Reserved SQL keywords used as table names can cause issues in raw queries. `prisma migrate reset` drops, recreates and migrates (it does not seed). https://www.prisma.io/docs/orm/v7/prisma-client/queries/crud https://www.prisma.io/docs/cli/v7/migrate/reset

## Per-pull-request databases in CI

- Provision a database when a pull request opens, apply the schema and seed, delete it when the pull request closes. Use a `concurrency` group with `cancel-in-progress: true`, sanitise the database name, and check that secrets are non-empty before use. https://www.prisma.io/docs/guides/v7/integrations/github-actions
