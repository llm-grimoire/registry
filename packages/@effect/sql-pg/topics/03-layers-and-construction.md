---
title: Layers and Construction
slug: layers-and-construction
description: How to create PgClient instances via make, fromPool, and Layer constructors
order: 3
category: setup
tags: [layer, construction, pool, scope]
relatedFiles: [src/PgClient.ts, test/utils.ts]
---

# Layers and Construction

## Overview

The package provides three layer constructors and two lower-level `Effect` constructors for creating `PgClient` instances. All layers provide both `PgClient` and `SqlClient` in the context.

## Key Concepts

### Layer Constructors

**`PgClient.layer(config)`** - Most common. Creates and manages a `pg.Pool` internally.
```typescript
const PgLive = PgClient.layer({
  database: "mydb",
  username: "postgres"
})
// Layer<PgClient | SqlClient, SqlError>
```

**`PgClient.layerConfig(config)`** - Same but reads config from Effect's `Config` system (environment variables, etc.).
```typescript
const PgLive = PgClient.layerConfig(
  Config.all({ url: Config.redacted("DATABASE_URL") })
)
// Layer<PgClient | SqlClient, ConfigError | SqlError>
```

**`PgClient.layerFromPool(options)`** - Uses an externally managed `pg.Pool`. You control pool lifecycle via `acquire`.
```typescript
const PgLive = PgClient.layerFromPool({
  acquire: Effect.acquireRelease(
    Effect.sync(() => new Pg.Pool({ connectionString: "..." })),
    (pool) => Effect.promise(() => pool.end())
  ),
  transformQueryNames: String.camelToSnake,
  transformResultNames: String.snakeToCamel
})
// Layer<PgClient | SqlClient, SqlError>
```

### What Layers Provide

All three layer constructors add **both** tags to the context:
```typescript
Context.make(PgClient, client).pipe(
  Context.add(Client.SqlClient, client)
)
```

This means downstream code can depend on the generic `SqlClient` for portable SQL code, or on `PgClient` for Postgres-specific features.

### Reactivity Layer

All layers automatically include `Reactivity.layer` (from `@effect/experimental`). No need to provide it separately.

### Lower-Level Constructors

**`PgClient.make(config)`** - Returns `Effect<PgClient, SqlError, Scope | Reactivity>`. Creates a pool, runs `SELECT 1` health check, sets up cleanup on scope finalization.

**`PgClient.fromPool(options)`** - Returns `Effect<PgClient, SqlError, Scope | Reactivity>`. For existing pools; does not manage pool lifecycle (the `acquire` effect should handle that).

### Startup Behavior

`make` does the following on construction:
1. Creates a `Pg.Pool` from config
2. Registers an error handler on the pool (swallows errors to prevent unhandled rejections)
3. Runs `SELECT 1` with a timeout (default 5s) to verify connectivity
4. Registers pool cleanup (`pool.end()`) as a scope finalizer
5. Parses the connection string (if URL was provided) to populate missing config fields

## Code Examples

Test setup with testcontainers:
```typescript
// From test/utils.ts - dynamic container-based layer
static ClientLive = Layer.unwrapEffect(
  Effect.gen(function*() {
    const container = yield* PgContainer
    return PgClient.layer({
      url: Redacted.make(container.getConnectionUri())
    })
  })
).pipe(Layer.provide(PgContainer.Default))
```

From-pool pattern (when you need pool control):
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
    return PgClient.layerFromPool({
      acquire,
      transformResultNames: String.snakeToCamel,
      transformQueryNames: String.camelToSnake
    })
  })
).pipe(Layer.provide(PgContainer.Default))
```

## Related Files

- `src/PgClient.ts` - `make`, `fromPool`, `layer`, `layerConfig`, `layerFromPool`
- `test/utils.ts` - Three layer variants used in tests
