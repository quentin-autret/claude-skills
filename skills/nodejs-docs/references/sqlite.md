# node:sqlite

Release candidate (Stability 1.2), added v22.5.0, without a flag since v23.4.0 (v22.13.0). Only the `node:` scheme works. https://nodejs.org/docs/latest/api/sqlite.html

## Contents

- Basics
- Constructor defaults
- Binding and SQL injection
- Type conversion
- Version-flagged APIs
- Old patterns

## Basics

- Every `Database` and `Statement` call is synchronous and blocks the event loop. https://nodejs.org/docs/latest/api/sqlite.html#class-database

```js
const { Database } = require('node:sqlite');
const db = new Database(':memory:');
db.exec('CREATE TABLE data(key INTEGER PRIMARY KEY, value TEXT) STRICT');
using insert = db.prepare('INSERT INTO data (key, value) VALUES (?, ?)');
insert.run(1, 'hello');
using query = db.prepare('SELECT * FROM data ORDER BY key');
console.log(query.all());
```
https://nodejs.org/docs/latest/api/sqlite.html

- Close statements once unused. `statement.close()` and `Symbol.dispose` (so `using`) are added in v26.8.0, so `using` on statements needs v26.8.0 or later. https://nodejs.org/docs/latest/api/sqlite.html
- The main classes are `Database` and `Statement`. `DatabaseSync` and `StatementSync` remain as deprecated aliases (renamed in v26.11.0, so v24 code still uses the old names). https://nodejs.org/docs/latest/api/sqlite.html#class-database , https://nodejs.org/docs/latest/api/deprecations.html#dep0210-sqlitedatabasesync
- SQLite file access is not covered by Permission Model fs grants. https://nodejs.org/docs/latest/api/permissions.html#file-system-permissions
- SQL trace events are published on the `diagnostics_channel` event `'sqlite.db.query'` (added v26.8.0, Stability 1). https://nodejs.org/docs/latest/api/diagnostics_channel.html#event-sqlitedbquery

## Constructor defaults

From https://nodejs.org/docs/latest/api/sqlite.html#new-databasepath-options

| Option | Default | Note |
| :-- | :-- | :-- |
| `timeout` | 0 | a locked database errors at once: set a busy timeout for concurrent access |
| `enableForeignKeyConstraints` | `true` | keep it on |
| `enableDoubleQuotedStringLiterals` | `false` | enabling is not recommended |
| `defensive` | `true` since v25.5.0 (v24.14.0) | blocks SQL that can corrupt the file |
| `allowExtension` | `false` | `loadExtension()` is off |
| `readBigInts` | `false` | large integers throw instead |
| `allowUnknownNamedParameters` | `false` | unknown keys throw `ERR_INVALID_STATE` |
| `readOnly` | `false` | read-only fails if the file is missing |
| `limits` | SQLite defaults | cap `length`, `sqlLength`, `exprDepth`... for untrusted input |

## Binding and SQL injection

- Bind user input as parameters, never interpolate it. https://nodejs.org/docs/latest/api/sqlite.html#class-statement
- Named parameters use `$`, `:` or `@` and bind from an object, bare keys allowed. Never mix numbered (`?NNN`) and named parameters: they share indexes. Unbound parameters are `NULL`. https://nodejs.org/docs/latest/api/sqlite.html#binding-parameters

```js
db.prepare('SELECT $a AS a, $b AS b').get({ $a: 1, $b: 2 });
db.prepare('SELECT ?2 AS a, ?1 AS b').get('first', 'second');
```

- `db.createTagStore()` (added v24.9.0) caches statements behind a tagged template. In the tagged form `${value}` is bound. In a plain template literal it is string interpolation, an injection: https://nodejs.org/docs/latest/api/sqlite.html#databasecreatetagstoremaxsize

```js
sql.run`INSERT INTO t1 (id) VALUES (${id})`; // bound, safe
db.exec(`INSERT INTO t1 (id) VALUES (${id})`); // interpolated, unsafe
```

## Type conversion

- `undefined` (since v26.10.0) and `null` bind as `NULL`. Booleans (bindable since v26.8.0) are stored as 1/0 and read back as numbers. Typed arrays and ArrayBuffers bind as BLOB and read back as `Uint8Array`. `NULL` reads as `null`, never `undefined`. https://nodejs.org/docs/latest/api/sqlite.html#type-conversion-between-javascript-and-sqlite
- An INTEGER beyond `Number.MAX_SAFE_INTEGER` throws `ERR_OUT_OF_RANGE` unless `readBigInts` is on. A bigint outside 64 bits throws `ERR_INVALID_ARG_VALUE`. https://nodejs.org/docs/latest/api/sqlite.html#type-conversion-between-javascript-and-sqlite

## Version-flagged APIs

| API | Added | Source |
| :-- | :-- | :-- |
| `db.serialize()` / `db.deserialize(buffer)` (replaces the database and finalizes its statements) | v26.1.0 | https://nodejs.org/docs/latest/api/sqlite.html#databaseserializedbname |
| `prepare(sql, { persistent })`, `statement.stat()`, `statement.close()` | v26.8.0 | https://nodejs.org/docs/latest/api/sqlite.html#databasepreparesql-options |
| `db.createModule()` (virtual tables) | v26.11.0 | https://nodejs.org/docs/latest/api/sqlite.html |
| `db.setAuthorizer()` | v24.10.0 | https://nodejs.org/docs/latest/api/sqlite.html |
| `db.createSession()`, `applyChangeset()` | v23.3.0 (v22.12.0) | https://nodejs.org/docs/latest/api/sqlite.html#databasecreatesessionoptions |

- `prepare('')` with no statement throws `ERR_INVALID_ARG_VALUE` since v26.8.0. https://nodejs.org/docs/latest/api/sqlite.html#databasepreparesql-options

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `new DatabaseSync(path)` | `new Database(path)` (v26.11.0+) | https://nodejs.org/docs/latest/api/deprecations.html#dep0210-sqlitedatabasesync |
| `StatementSync` | `Statement` | https://nodejs.org/docs/latest/api/deprecations.html#dep0211-sqlitestatementsync |
| `--experimental-sqlite` | not needed | https://nodejs.org/docs/latest/api/sqlite.html |
