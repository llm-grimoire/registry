---
title: SQL Statements and Compiler
slug: sql-statements-compiler
description: Tagged template SQL, the Pg compiler, helpers (insert, update, in, and), and custom types
order: 4
category: api
tags: [sql, compiler, statement, insert, update, query]
relatedFiles: [src/PgClient.ts, test/Client.test.ts]
---

# SQL Statements and Compiler

## Overview

`PgClient` uses `@effect/sql`'s tagged template literal system for SQL composition. The Pg-specific compiler converts statements to PostgreSQL syntax with `$1`-style placeholders, identifier escaping with double quotes, and support for the `PgJson` custom type.

## Key Concepts

### Tagged Template SQL

The client itself is a tagged template function:
```typescript
const sql = yield* PgClient.PgClient

// Simple query
const rows = yield* sql`SELECT * FROM people WHERE id = ${id}`

// Identifier escaping
sql`SELECT * FROM ${sql("tableName")}` // → "tableName"
```

### SQL Helpers

**`sql.insert(record)`** - Generates INSERT column/value lists:
```typescript
sql`INSERT INTO people ${sql.insert({ name: "Tim", age: 10 })}`
// → INSERT INTO people ("name","age") VALUES ($1,$2)
```

**`sql.update(record, omit?)`** - Generates SET clauses:
```typescript
sql`UPDATE people SET ${sql.update({ name: "Tim", age: 10 }, ["age"])}`
// → UPDATE people SET "name" = $1
// (age is omitted)
```

**`sql.updateValues(records, alias)`** - Generates VALUES with alias for bulk updates:
```typescript
sql`UPDATE people SET name = data.name FROM ${
  sql.updateValues([{ name: "Tim" }, { name: "John" }], "data")
}`
// → UPDATE people SET name = data.name FROM (values ($1),($2)) AS data("name")
```

Both `update` and `updateValues` support `.returning("*")`:
```typescript
sql.updateValues([...], "data").returning("*")
// → ... RETURNING *
```

**`sql.in(values)` / `sql.in(column, values)`** - IN clause:
```typescript
sql`WHERE id IN ${sql.in([1, 2, 3])}`        // → WHERE id IN ($1,$2,$3)
sql`WHERE ${sql.in("id", [1, 2, 3])}`        // → WHERE "id" IN ($1,$2,$3)
sql`WHERE ${sql.in("id", [])}`               // → WHERE 1=0
```

**`sql.and(fragments)`** - AND composition:
```typescript
sql`WHERE ${sql.and([
  sql.in("name", ["Tim", "John"]),
  sql`created_at < ${now}`
])}`
// → WHERE ("name" IN ($1,$2) AND created_at < $3)
```

### JSON Handling

The `sql.json()` method wraps a value with the `PgJson` custom type:
```typescript
sql`INSERT INTO people ${sql.insert({
  name: "Tim",
  data: sql.json({ a: 1 })
})}`
```

PostgreSQL also handles inline JSON via `::jsonb` cast:
```typescript
const rows = yield* sql`SELECT ${{ testValue: 123 }}::jsonb AS json`
// rows[0].json → { testValue: 123 }
```

### Multi-Statement Queries

PostgreSQL supports multiple statements in a single query. Results come back as an array of arrays:
```typescript
const result = yield* sql`
  CREATE TABLE test (id TEXT PRIMARY KEY, name TEXT);
  INSERT INTO test VALUES ('1', 'a') RETURNING *;
  INSERT INTO test VALUES ('2', 'b') RETURNING *;
`
// result[0] → [] (CREATE has no rows)
// result[1] → [{ id: "1", name: "a" }]
// result[2] → [{ id: "2", name: "b" }]
```

### The Compiler

`PgClient.makeCompiler(transform?, transformJson?)` creates a `Statement.Compiler` with:
- **PostgreSQL `$N` placeholders** (not `?`)
- **Double-quote identifier escaping** (`"column_name"`)
- **Name transforms** applied to identifiers
- **`onRecordUpdate`** for VALUES-based bulk update syntax
- **`PgJson` custom type** handling

```typescript
const compiler = PgClient.makeCompiler(String.camelToSnake)
const [query] = compiler.compile(
  sql`SELECT * FROM ${sql("peopleTest")}`, false
)
// → SELECT * FROM "people_test"
```

### Disabling Transforms

Use `sql.withoutTransforms()` to get a client that skips name transforms:
```typescript
const raw = (yield* PgClient.PgClient).withoutTransforms()
raw`INSERT INTO people ${raw.insert({ first_name: "Tim" })}`
// Uses column names as-is, no camelToSnake
```

## Related Files

- `src/PgClient.ts` - `makeCompiler` function and `PgJson` custom type
- `test/Client.test.ts` - Comprehensive helper and compiler tests
