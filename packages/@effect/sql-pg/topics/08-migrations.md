---
title: Migrations
slug: migrations
description: Database migration support via PgMigrator wrapping @effect/sql/Migrator
order: 8
category: api
tags: [migrations, migrator, pg_dump, schema]
relatedFiles: [src/PgMigrator.ts]
---

# Migrations

## Overview

`PgMigrator` wraps `@effect/sql/Migrator` with a PostgreSQL-specific schema dump implementation. It uses `pg_dump` to capture the database schema after migrations run, enabling schema snapshots for version control.

## Key Concepts

### Module Exports

`PgMigrator` re-exports everything from `@effect/sql/Migrator` and `@effect/sql/Migrator/FileSystem`, plus adds:
- `run(options)` - Execute migrations
- `layer(options)` - Layer that runs migrations on construction

### The `run` Function

```typescript
export const run: <R2 = never>(
  options: Migrator.MigratorOptions<R2>
) => Effect.Effect<
  ReadonlyArray<readonly [id: number, name: string]>,
  Migrator.MigrationError | SqlError,
  FileSystem | Path | PgClient | Client.SqlClient | CommandExecutor | R2
>
```

Returns an array of `[id, name]` tuples for applied migrations.

### Schema Dump with pg_dump

The Postgres-specific `dumpSchema` implementation:

1. Runs `pg_dump --schema-only` for table/index/constraint definitions
2. Runs `pg_dump --column-inserts --data-only --table=<migrations_table>` for migration records
3. Combines both outputs
4. Cleans up the dump: removes comments (`--`), SET statements, and `pg_catalog` calls
5. Writes to the specified path

Connection details for `pg_dump` are read from the `PgClient.config`:
```typescript
Command.env({
  PATH: process.env.PATH,
  PGHOST: sql.config.host,
  PGPORT: sql.config.port?.toString(),
  PGUSER: sql.config.username,
  PGPASSWORD: sql.config.password ? Redacted.value(sql.config.password) : undefined,
  PGDATABASE: sql.config.database,
  PGSSLMODE: sql.config.ssl ? "require" : "prefer"
})
```

### Dependencies

`run` requires:
- `PgClient` and `SqlClient` - for database access and config
- `FileSystem` and `Path` (from `@effect/platform`) - for writing dump files
- `CommandExecutor` (from `@effect/platform`) - for running `pg_dump`

### Layer Constructor

```typescript
export const layer = <R>(
  options: Migrator.MigratorOptions<R>
): Layer.Layer<
  never,
  Migrator.MigrationError | SqlError,
  PgClient | Client.SqlClient | CommandExecutor | FileSystem | Path | R
>
```

Runs migrations as a side effect during layer construction. The layer provides `never` (no services) — it's used purely for its initialization effect.

## Code Examples

Running migrations:
```typescript
import { PgMigrator } from "@effect/sql-pg"

// As an effect
const applied = yield* PgMigrator.run({
  directory: "migrations",
  table: "schema_migrations"
})

// As a layer (runs on app startup)
const MigrationsLive = PgMigrator.layer({
  directory: "migrations"
})
```

## Related Files

- `src/PgMigrator.ts` - Complete migrator implementation
- `src/PgClient.ts` - `PgClient.config` used for `pg_dump` connection details
