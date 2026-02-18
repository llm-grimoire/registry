---
title: Connection Management and Pooling
slug: connection-management
description: How connections are acquired, released, and pooled via pg.Pool
order: 5
category: architecture
tags: [pool, connection, transaction, scope, cancellation]
relatedFiles: [src/PgClient.ts]
---

# Connection Management and Pooling

## Overview

`@effect/sql-pg` uses `pg.Pool` for connection management. The internal `ConnectionImpl` class wraps pool-acquired connections with proper cleanup, error handling, and query cancellation support. Transactions use dedicated connections reserved via `Effect.Scope`.

## Key Concepts

### Two Modes of Connection Use

**Pooled (default)**: Each query acquires a connection from the pool, executes, and releases it. This is the `acquirer` path — `new ConnectionImpl()` (no pre-assigned client).

**Reserved (transactions)**: A dedicated `Pg.PoolClient` is acquired and held for the scope of a transaction. This is the `transactionAcquirer` path — `new ConnectionImpl(client)`.

### ConnectionImpl Internal

The `ConnectionImpl` class implements the `Connection` interface from `@effect/sql`. Its `runWithClient` method handles both modes:

- **With pre-assigned client** (`this.pg !== undefined`): Uses it directly, supports cancellation.
- **Without client**: Acquires from pool via `pool.connect()`, registers error handler, executes query, then releases.

```typescript
// Simplified flow for pooled queries
pool.connect((cause, client) => {
  client.once("error", onError)
  // execute query
  // on complete: cleanup (release client, remove error handler)
})
```

### Query Cancellation

When an Effect fiber is interrupted, in-flight queries are cancelled via PostgreSQL's `pg_cancel_backend()`:

```typescript
const makeCancel = (pool: Pg.Pool, client: Pg.PoolClient) => {
  const processId = (client as any).processID
  // Sends: SELECT pg_cancel_backend(<processId>)
  // Best-effort, 5 second timeout
}
```

Cancel effects are cached per-client via a `WeakMap` to avoid creating duplicates.

### Transaction Connections

Transaction connections (`reserve`) are acquired differently:
1. A `Pg.PoolClient` is acquired and held (not released after each query)
2. An error handler is registered on the client
3. A scope finalizer releases the client when the transaction scope ends
4. The error cause is passed to `release()` so the pool knows to destroy the connection if it errored

```typescript
// Reserving a connection for transaction use
const reserve = Effect.map(reserveRaw, (client) =>
  new ConnectionImpl(client)
)
```

Consumers use it via `sql.withTransaction`:
```typescript
yield* sql.withTransaction(
  Effect.all([
    Insert.execute({ name: "John" }),
    Insert.execute({ name: "Jane" })
  ], { batching: true })
)
```

Or via `sql.reserve` for manual connection control:
```typescript
const conn = yield* sql.reserve
yield* conn.executeRaw("select pg_sleep(1000)", [])
```

### Interruption Support

The test suite verifies that queries can be interrupted and the connection remains usable:

```typescript
// From test/Client.test.ts
const conn = yield* sql.reserve
yield* conn.executeRaw("select pg_sleep(1000)", []).pipe(
  Effect.timeoutOption("50 millis"),
  TestServices.provideLive
)
// Connection still works after interruption
const value = yield* conn.executeValues("select 1", [])
```

### LISTEN/NOTIFY Connection

A separate connection is managed via `RcRef` for LISTEN/NOTIFY. This ensures the listening connection stays open and isn't returned to the pool between notifications. The `RcRef` is reference-counted and lazily acquired.

## Related Files

- `src/PgClient.ts` - `ConnectionImpl`, `makeCancel`, `reserveRaw`, `reserve`, `listenClient`
- `test/Client.test.ts` - Interruption test showing cancel behavior
