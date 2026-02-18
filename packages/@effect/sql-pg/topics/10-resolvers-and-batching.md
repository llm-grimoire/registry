---
title: SQL Resolvers and Batching
slug: resolvers-and-batching
description: Batched database access using SqlResolver with Effect's request batching
order: 10
category: patterns
tags: [resolver, batching, SqlResolver, request]
relatedFiles: [examples/resolver.ts]
---

# SQL Resolvers and Batching

## Overview

`@effect/sql` provides `SqlResolver` for defining batched database access patterns. When used with `{ batching: true }` in `Effect.all` or `Effect.forEach`, multiple individual requests are automatically combined into single SQL queries. The resolver example demonstrates this with the Pg client.

## Key Concepts

### Resolver Types

**`SqlResolver.ordered`** - For insert/update operations where result order matches request order:
```typescript
const Insert = yield* SqlResolver.ordered("InsertPerson", {
  Request: InsertPersonSchema,
  Result: Person,
  execute: (requests) =>
    sql`INSERT INTO people ${sql.insert(requests)} RETURNING people.*`
})
```

**`SqlResolver.findById`** - For lookup by unique ID:
```typescript
const GetById = yield* SqlResolver.findById("GetPersonById", {
  Id: Schema.Number,
  Result: Person,
  ResultId: (result) => result.id,
  execute: (ids) =>
    sql`SELECT * FROM people WHERE id IN ${sql.in(ids)}`
})
```

**`SqlResolver.grouped`** - For lookup where multiple results can match each request:
```typescript
const GetByName = yield* SqlResolver.grouped("GetPersonByName", {
  Request: Schema.String,
  RequestGroupKey: (_) => _,
  Result: Person,
  ResultGroupKey: (_) => _.name,
  execute: (ids) =>
    sql`SELECT * FROM people WHERE name IN ${sql.in(ids)}`
})
```

### Batching in Action

Individual calls are batched into single queries:
```typescript
// These two execute as ONE SQL statement
const [person1, person2] = yield* Effect.all(
  [Insert.execute({ name: "John" }), Insert.execute({ name: "Jane" })],
  { batching: true }
)

// These three lookups execute as ONE SQL statement
yield* Effect.forEach(
  ["John", "Jane", "John"],
  (name) => GetByName.execute(name),
  { batching: true }
)
```

### Schema Integration

Resolvers use `effect/Schema` for request/result validation:
```typescript
class Person extends Schema.Class<Person>("Person")({
  id: Schema.Number,
  name: Schema.String,
  createdAt: Schema.DateFromSelf
}) {}

const InsertPersonSchema = Schema.Struct(Person.fields).pipe(
  Schema.omit("id", "createdAt")
)
```

### Transaction Support

Resolvers work within transactions:
```typescript
yield* sql.withTransaction(
  Effect.all(
    [Insert.execute({ name: "John" }), Insert.execute({ name: "Jane" })],
    { batching: true }
  )
)
```

## Related Files

- `examples/resolver.ts` - Complete resolver example with ordered, findById, and grouped patterns
- `src/PgClient.ts` - SQL helpers (`insert`, `in`) used in resolver execute functions
