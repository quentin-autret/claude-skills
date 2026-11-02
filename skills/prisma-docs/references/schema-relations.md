# Schema: relations, referential actions and relation mode

## Contents
- Relation fields
- One-to-one and one-to-many
- Many-to-many
- Self-relations
- Referential actions
- Relation mode

## Relation fields

- A relation has exactly 2 relation fields, one per model. The side with `@relation(fields, references)` holds the foreign key scalar. Relation fields exist only in Prisma. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations
  ```prisma
  model Post {
    id       Int  @id @default(autoincrement())
    author   User @relation(fields: [authorId], references: [id], onDelete: Cascade)
    authorId Int
  }
  ```
  https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations
- `references` targets an `@id` or `@unique` (or a compound `@@id`/`@@unique`). `fields` and `references` list the same number of fields in the same order. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations
- Two relations between the same models need a `name`, identical on both sides: `@relation("WrittenPosts")`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations
- Define both sides of every relation. https://www.prisma.io/docs/orm/v7/more/best-practices

## One-to-one and one-to-many

- One-to-one: put `@unique` on the foreign key scalar, otherwise it is really one-to-many. The side without the scalar must be optional (`Profile?`). You choose which side stores the key. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/one-to-one-relations
- One-to-many: the list side (`Post[]`) is always required. Whether the parent is mandatory is decided by the key side (`User`/`Int` or `User?`/`Int?`). https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/one-to-many-relations

## Many-to-many

- Prefer implicit (`categories Category[]` and `posts Post[]`, no `@relation`). Use implicit unless you need to store additional metadata in the relation table, which takes an explicit join model. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/many-to-many-relations
- Implicit rules: both models have a single-field `@id`. No `fields`, `references`, `onDelete` or `onUpdate`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/many-to-many-relations
- Implicit table convention, needed for `db pull` to recognise it: `_CategoryToPost` (model names in alphabetical order), columns `A` and `B`. A custom table name uses the same `@relation("MyName")` on both sides. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/many-to-many-relations
- Different names on the two sides of an implicit relation make Prisma expect two one-to-many relations. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/troubleshooting-relations
- Implicit join tables have no primary key. Some cloud providers enforce a primary key on every table, which then requires an explicit model with a primary key. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/troubleshooting-relations
- Explicit: a join model with 2 relations and `@@id([postId, categoryId])`. Both parents list the join model (`categories CategoriesOnPosts[]`), not each other. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/many-to-many-relations
- Converting implicit to explicit: add the join model next to the implicit relation and migrate, copy the rows with a script, then remove the implicit fields and migrate (drops the implicit table). https://www.prisma.io/docs/orm/v7/more/troubleshooting/many-to-many-relations

## Self-relations

- Always need `@relation` with a shared name. One-to-one: one side holds `fields/references` with `@unique`, both sides cannot be required. One-to-many: no `@unique`. Explicit many-to-many: two named relations and `@@id([followingId, followedById])`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/self-relations
- In an implicit many-to-many self-relation, columns `A` and `B` follow the alphabetical order of the relation field names. Renaming fields so that order flips silently swaps the meaning of existing rows. Keep the order stable (prefix `a_`, `b_`). https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/troubleshooting-relations

## Referential actions

- `onDelete` and `onUpdate` on `@relation`: `Cascade`, `Restrict`, `NoAction`, `SetNull`, `SetDefault`. Defaults: `onDelete` is `SetNull` for optional relations and `Restrict` for required ones, `onUpdate` is `Cascade`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/referential-actions
- `SetNull` needs an optional relation and scalar, otherwise a runtime error (PostgreSQL only warns, others reject the schema). `SetDefault` needs a `@default` on the scalar and is not really supported by MySQL/MariaDB. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/referential-actions
- `NoAction` differs: PostgreSQL can defer the check to the end of the transaction, MySQL treats it as `Restrict`, SQLite does nothing. SQL Server has no `Restrict`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/referential-actions
- SQL Server rejects cycles and multiple cascade paths (MongoDB also needs `NoAction` for self-relations and 3-model cycles). Break the cycle by setting `NoAction` on one relation of it: the page's examples use `onUpdate: NoAction`, and the self-relation example sets both `onDelete` and `onUpdate`. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/referential-actions
- Implicit many-to-many relations take no referential actions. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/referential-actions
- Deleting a parent that has children through a required relation fails with a "would violate the required relation" error. Alternatives: make the relation optional, reassign the children first, or `$transaction([deleteMany children, delete parent])`. https://www.prisma.io/docs/orm/v7/prisma-client/queries/crud

## Relation mode

- Default `relationMode = "foreignKeys"` (real constraints). Use `"prisma"` (the only mode on MongoDB) when the database does not support foreign keys, because emulation costs extra queries and foreign keys are usually preferred when available. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/relation-mode
- Emulation covers Prisma Client queries only, not raw SQL. On create no constraint is checked, so invalid references can be written. Emulated actions: `Cascade`, `Restrict`, `SetNull` everywhere. `NoAction` is not supported on PostgreSQL and SQLite (use `Restrict`) and, where it is accepted, creates no check that protects integrity. `SetDefault` is never emulated. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/relation-mode
- No foreign key means no implicit index: add `@@index` on each relation scalar (`prisma validate` warns). `db pull` does not produce relations in this mode: add `@relation` by hand once, later pulls keep it. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/relation-mode
- Switching mode adds or drops foreign keys on the next migration. When moving to a database without foreign keys, the docs recommend deleting the migrations folder, since the existing history contains foreign key DDL that may error if rerun. https://www.prisma.io/docs/orm/v7/prisma-schema/data-model/relations/relation-mode
