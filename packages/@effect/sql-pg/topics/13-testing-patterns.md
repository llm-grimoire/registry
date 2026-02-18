---
title: Testing Patterns
slug: testing-patterns
description: Test setup with testcontainers, layer composition, and test utilities
order: 13
category: guides
tags: [testing, testcontainers, vitest, layers]
relatedFiles: [test/utils.ts, test/Client.test.ts, vitest.config.ts]
---

# Testing Patterns

## Overview

The package tests use `@testcontainers/postgresql` to spin up real PostgreSQL instances in Docker. The test utilities demonstrate idiomatic Effect layer composition for test environments.

## Key Concepts

### PgContainer Service

`test/utils.ts` defines a `PgContainer` service using `Effect.Service`:

```typescript
export class PgContainer extends Effect.Service<PgContainer>()("test/PgContainer", {
  scoped: Effect.acquireRelease(
    Effect.tryPromise({
      try: () => new PostgreSqlContainer("postgres:alpine").start(),
      catch: (cause) => new ContainerError({ cause })
    }),
    (container) => Effect.promise(() => container.stop())
  )
}) {}
```

The container is started once and shared across tests via the layer system.

### Three Test Layer Variants

**`ClientLive`** - Basic client from container URL:
```typescript
static ClientLive = Layer.unwrapEffect(
  Effect.gen(function*() {
    const container = yield* PgContainer
    return PgClient.layer({
      url: Redacted.make(container.getConnectionUri())
    })
  })
).pipe(Layer.provide(PgContainer.Default))
```

**`ClientTransformLive`** - Client with snake/camel transforms:
```typescript
static ClientTransformLive = Layer.unwrapEffect(
  Effect.gen(function*() {
    const container = yield* PgContainer
    return PgClient.layer({
      url: Redacted.make(container.getConnectionUri()),
      transformResultNames: String.snakeToCamel,
      transformQueryNames: String.camelToSnake
    })
  })
).pipe(Layer.provide(PgContainer.Default))
```

**`ClientFromPoolLive`** - Client from external pool:
```typescript
static ClientFromPoolLive = Layer.unwrapEffect(
  Effect.gen(function*() {
    const container = yield* PgContainer
    const acquire = Effect.acquireRelease(
      Effect.sync(() => new Pg.Pool({
        connectionString: container.getConnectionUri()
      })),
      (pool) => Effect.promise(() => pool.end())
    )
    return PgClient.layerFromPool({ acquire, ...transforms })
  })
).pipe(Layer.provide(PgContainer.Default))
```

### Test Structure with @effect/vitest

Tests use `it.layer()` to provide the test layer, then `it.effect()` for individual Effect-based tests:

```typescript
it.layer(PgContainer.ClientLive, { timeout: "30 seconds" })("PgClient", (it) => {
  it.effect("insert helper", () =>
    Effect.gen(function*() {
      const sql = yield* PgClient.PgClient
      const [query, params] = sql`INSERT INTO people ${
        sql.insert({ name: "Tim", age: 10 })
      }`.compile()
      expect(query).toEqual(`INSERT INTO people ("name","age") VALUES ($1,$2)`)
    }))
})
```

### TaggedError for Container Failures

```typescript
export class ContainerError extends Data.TaggedError("ContainerError")<{
  cause: unknown
}> {}
```

### Vitest Configuration

The vitest config is minimal, extending a shared monorepo config:
```typescript
// vitest.config.ts merges with ../../vitest.shared.js
```

## Related Files

- `test/utils.ts` - `PgContainer` service and three layer variants
- `test/Client.test.ts` - Comprehensive test suite
- `test/SqlPersistedQueue.test.ts` - Shared test suite from `@effect/sql`
- `vitest.config.ts` - Test runner configuration
