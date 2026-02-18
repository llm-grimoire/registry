---
title: Package Overview
slug: overview
description: What @effect/sql-pg is and how it fits into the Effect SQL ecosystem
order: 0
category: architecture
tags: [overview, postgresql, effect-sql]
relatedFiles: [src/index.ts, src/PgClient.ts, package.json]
---

# Package Overview

## Overview

`@effect/sql-pg` is the PostgreSQL implementation for the `@effect/sql` abstraction layer. It wraps the `pg` (node-postgres) library and its ecosystem (`pg-pool`, `pg-cursor`, `pg-types`, `pg-connection-string`) to provide a fully Effect-native PostgreSQL client with connection pooling, streaming, LISTEN/NOTIFY, migrations, and SQL statement compilation.

The package lives in the Effect monorepo at `packages/sql-pg` and exports exactly two modules: `PgClient` and `PgMigrator`.

## Key Concepts

- **Part of `@effect/sql` family**: Implements the `SqlClient` interface from `@effect/sql`, meaning any code written against the generic `SqlClient` tag works with Postgres transparently.
- **Built on `pg` (node-postgres)**: Uses `pg.Pool` for connection management, `pg-cursor` for streaming, `pg-connection-string` for URL parsing.
- **Effect-native**: All operations return `Effect` values. Connections, pools, and listeners are managed through `Scope` and `Layer`.
- **Two modules total**: `PgClient` (client, config, layers, compiler) and `PgMigrator` (migration runner wrapping `@effect/sql/Migrator`).

## Package Exports

```typescript
// src/index.ts
export * as PgClient from "./PgClient.js"
export * as PgMigrator from "./PgMigrator.js"
```

Consumer imports:
```typescript
import { PgClient, PgMigrator } from "@effect/sql-pg"
```

## Peer Dependencies

The package requires these Effect ecosystem packages as peer dependencies:
- `effect` - core runtime
- `@effect/sql` - SQL abstraction layer (SqlClient, Statement, SqlError, Migrator)
- `@effect/platform` - FileSystem, Path, Command (used by migrator)
- `@effect/experimental` - Reactivity service (used for reactive queries)

## Related Files

- `src/index.ts` - Package entry point, re-exports both modules
- `src/PgClient.ts` - Core client implementation (~470 lines)
- `src/PgMigrator.ts` - Migration support (~80 lines)
- `package.json` - Dependencies and build configuration
