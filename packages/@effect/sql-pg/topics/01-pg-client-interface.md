---
title: PgClient Interface and Tag
slug: pg-client-interface
description: The PgClient type, its capabilities beyond SqlClient, and the Context tag
order: 1
category: api
tags: [PgClient, interface, tag, SqlClient]
relatedFiles: [src/PgClient.ts]
---

# PgClient Interface and Tag

## Overview

`PgClient` is the central type in the package. It extends `SqlClient` (from `@effect/sql`) with PostgreSQL-specific capabilities: JSON fragment handling, LISTEN/NOTIFY streaming, and access to the resolved connection config.

## Key Concepts

### The Interface

```typescript
export interface PgClient extends Client.SqlClient {
  readonly [TypeId]: TypeId
  readonly config: PgClientConfig
  readonly json: (_: unknown) => Fragment
  readonly listen: (channel: string) => Stream.Stream<string, SqlError>
  readonly notify: (channel: string, payload: string) => Effect.Effect<void, SqlError>
}
```

Beyond the inherited `SqlClient` capabilities (tagged template SQL, `insert`, `update`, `updateValues`, `in`, `and`, `withTransaction`, `reserve`, `onDialect`, `stream`), `PgClient` adds:

- **`config`** - The resolved `PgClientConfig` (host, port, database, etc.). Populated from explicit config and parsed from connection URL.
- **`json`** - Creates a `Fragment` for JSON parameters, handled by the PgJson custom type in the compiler.
- **`listen`** - Returns a `Stream<string, SqlError>` of LISTEN/NOTIFY payloads for a channel.
- **`notify`** - Sends a NOTIFY with a payload to a channel.

### The Context Tag

```typescript
export const PgClient = Context.GenericTag<PgClient>("@effect/sql-pg/PgClient")
```

All layers provide both `PgClient` and `Client.SqlClient` in the context, so consumers can depend on either:

```typescript
// Postgres-specific features (listen/notify, json, config)
const sql = yield* PgClient.PgClient

// Generic SQL (works with any @effect/sql backend)
const sql = yield* SqlClient.SqlClient
```

### TypeId

The `TypeId` is a branded string `"~@effect/sql-pg/PgClient"` used for nominal typing, ensuring type safety when working with the PgClient interface.

## Code Examples

Accessing Pg-specific features:
```typescript
const sql = yield* PgClient.PgClient

// Use json fragment in insert
yield* sql`INSERT INTO people ${sql.insert({
  name: "Tim",
  data: sql.json({ a: 1 })
})}`

// Dialect dispatch
sql.onDialect({
  pg: () => "PostgreSQL",
  sqlite: () => "SQLite",
  mysql: () => "MySQL",
  mssql: () => "MSSQL",
  clickhouse: () => "ClickHouse"
})
```

## Related Files

- `src/PgClient.ts` - Full interface definition (lines 1-70)
