# Client queries: CRUD, shaping, relations, filters, aggregation

## Contents
- Client construction options
- CRUD and bulk operations
- select, include, omit
- Nested writes
- Relation queries and load strategy
- Filters and sorting
- Pagination
- Aggregation, groupBy, distinct
- Performance

## Client construction options

- `adapter` (a driver adapter) is required unless `accelerateUrl` is set. Other options: `log`, `errorFormat`, `omit`, `transactionOptions`, `comments`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#adapter
- Global omit hides sensitive fields everywhere: `new PrismaClient({ adapter, omit: { user: { password: true } } })`. https://www.prisma.io/docs/orm/v7/more/best-practices
- A query is a lazy `PrismaPromise`: it runs only on `await`, `.then()` or `.catch()`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#prismapromise-behavior

## CRUD and bulk operations

- `update` and `delete` throw `P2025` when the record is missing. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#update
- `findUniqueOrThrow` and `findFirstOrThrow` throw `PrismaClientKnownRequestError`. `findFirst` with a negative `take` reverses the order. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#findfirst
- `findUnique` cannot use filter conditions (`equals`, `contains`, `not`) on `Json` fields, which likely returns `null` for that field. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#findunique
- `where` of `findUnique`, `update`, `delete`, `upsert` accepts extra non-unique fields, as long as one unique field sits outside `AND/OR/NOT`. Useful for permission checks and soft deletes: https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#filter-on-non-unique-fields-with-userwhereuniqueinput
  ```ts
  await prisma.post.update({ where: { id: 1, authorId: userId }, data });
  ```
  https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#permission-checks
- `updateMany` and `deleteMany` without `where` touch every row. Both accept `limit`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#updatemany
- Default for batches: `createMany` (returns `{ count }`), not many separate `create` calls. Split very large inserts into batches (the docs give 50,000 records in batches of 1000). https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#create https://www.prisma.io/docs/orm/v7/prisma-client/queries/advanced/query-optimization-performance
- `skipDuplicates` is not supported on MongoDB, SQL Server or SQLite. `createManyAndReturn` and `updateManyAndReturn` exist on PostgreSQL, CockroachDB and SQLite only. https://www.prisma.io/docs/orm/v7/prisma-client/queries/crud
- `createManyAndReturn` does not guarantee result order, issues one query per included relation and does not support `relationLoadStrategy: "join"`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#createmanyandreturn
- Top-level `createMany` cannot create relations. Create parents with `createManyAndReturn({ select: { id: true } })`, then children with the returned ids. https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
- Find-or-create: `upsert` with `update: {}`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/crud
- Atomic number operations: `increment`, `decrement`, `multiply`, `divide`, `set`, one per field per query. They leave a `null` field unchanged. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#atomic-number-operations

## select, include, omit

- `select` and `include` cannot sit at the same level. Nest instead: https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
  ```ts
  select: { email: true, posts: { select: { title: true } } }
  ```
  https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
- `select` cannot combine with `omit` in the same query. https://www.prisma.io/docs/orm/v7/more/best-practices
- Default: `omit` when the default shape fits minus a few fields, `select` when keeping only a few. https://www.prisma.io/docs/orm/v7/prisma-client/queries/excluding-fields
- Relation counts: `select: { _count: { select: { posts: { where: { title: "Hello!" } } } } }`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/aggregation-grouping-summarizing
- Reusable typed args: `const omitPassword = { password: true } satisfies Prisma.UserOmit`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#generated-types-for-omit

## Nested writes

- Nested writes run in one transaction. https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
- `connect` throws when a target is missing. Use `connectOrCreate` (`where` on a unique field plus `create`). https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#connect
- `set` replaces the whole list. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#set With `set` and `connect` together, order matters: `set` then `connect` keeps only the connected records, `connect` then `set` is overridden by `set`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#connect
- `disconnect` exists on optional relations only. Clear a one-to-many list with `set: []`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#disconnect https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
- Nested `createMany` works on one-to-many only, without further nesting. Default to nested `create` unless `skipDuplicates` is needed. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#createmany-1
- Nested `update` throws when the parent or related record is missing. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#update-1

## Relation queries and load strategy

- To-many filters: `some`, `every`, `none` (`none: {}` = no related rows). To-one: `is`, `isNot`, or `author: null`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
- Fluent API `findUnique(...).posts()` runs 2 queries, and `findMany().posts()` is illegal. https://www.prisma.io/docs/orm/v7/prisma-client/queries/relation-queries
- `relationLoadStrategy: "join"` (default; one query, LATERAL JOIN on PostgreSQL, correlated subqueries on MySQL) or `"query"` (one query per table). It is Preview: enable `relationJoins` (PostgreSQL, CockroachDB, MySQL). Top level only. `query` can win on some datasets, so profile. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#relationloadstrategy-preview
- Explicit many-to-many reads: `include: { tags: { include: { tag: true } } }`, then `post.tags.map((t) => t.tag)`. https://www.prisma.io/docs/orm/v7/more/troubleshooting/many-to-many-relations

## Filters and sorting

- `not` never returns rows where a nullable column is NULL. Include them explicitly: `OR: [{ name: { not: "Eleanor" } }, { name: null }]`. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#not
- `mode: "insensitive"` is supported on PostgreSQL and MongoDB only. https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#mode
- Compare two columns of one model with `fields`: https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#compare-columns-in-the-same-table
  ```ts
  prisma.product.findMany({ where: { quantity: { lte: prisma.product.fields.warnQuantity } } });
  ```
  https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#compare-columns-in-the-same-table
- `orderBy` with `nulls: "first" | "last"` works only on optional fields (`P2009` otherwise). https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#orderby
- Full-text search on PostgreSQL needs the `fullTextSearchPostgres` preview: `where: { body: { search: "cat | dog" } }`, sorted by `_relevance`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/full-text-search

## Pagination

- Offset (`skip`/`take`) for numbered pages and shallow navigation, its cost grows with the offset. Default for feeds and large sets: cursor with `take`, `skip: 1`, `cursor: { id: lastId }` and a stable `orderBy`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/pagination

## Aggregation, groupBy, distinct

- `groupBy` returns every `by` field and allows no `select`. `where` filters rows before grouping (prefer it), `having` filters groups and may use only aggregates or `by` fields. https://www.prisma.io/docs/orm/v7/prisma-client/queries/aggregation-grouping-summarizing
- `groupBy` with `skip`/`take` requires `orderBy`, on `by` fields or an aggregate. https://www.prisma.io/docs/orm/v7/prisma-client/queries/aggregation-grouping-summarizing
- Aggregates on nullable fields return `null` when there are no rows or every value is `null`, while `count` returns 0 when no record is found. https://www.prisma.io/docs/orm/v7/prisma-client/queries/aggregation-grouping-summarizing
- `distinct` is not SQL `SELECT DISTINCT`: Prisma selects then deduplicates in memory. The `nativeDistinct` preview pushes it to the database, where supported. https://www.prisma.io/docs/orm/v7/prisma-client/queries/aggregation-grouping-summarizing https://www.prisma.io/docs/orm/v7/reference/prisma-client-reference#nativedistinct

## Performance

- Avoid N+1: `include` or `where: { authorId: { in: ids } }` (2 queries), or `relationLoadStrategy: "join"` (1 query). https://www.prisma.io/docs/orm/v7/prisma-client/queries/advanced/query-optimization-performance
- `findUnique` calls in the same tick with the same parameters and selection set are batched into one query (dataloader). `findMany` is not batched. In GraphQL resolvers, use `findUnique({ where }).posts()`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/advanced/query-optimization-performance
