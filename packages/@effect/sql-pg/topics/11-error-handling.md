---
title: Error Handling
slug: error-handling
description: SqlError, MigrationError, and error patterns in the Pg client
order: 11
category: patterns
tags: [errors, SqlError, MigrationError]
relatedFiles: [src/PgClient.ts, src/PgMigrator.ts]
---

# Error Handling

## Overview

`@effect/sql-pg` uses two error types: `SqlError` (from `@effect/sql`) for all database operations, and `MigrationError` (from `@effect/sql/Migrator`) for migration-specific failures. All errors wrap the underlying `pg` library errors as `cause`.

## Key Concepts

### SqlError

All database operations fail with `SqlError`, which wraps the original error:

```typescript
new SqlError({ cause: pgError, message: "Failed to execute statement" })
```

Common error messages used in the codebase:
- `"PgClient: Failed to connect"` - Initial `SELECT 1` health check failed
- `"PgClient: Connection timed out"` - Connect timeout exceeded
- `"Failed to acquire connection"` - Pool couldn't provide a client
- `"Connection error"` - Error on an active connection
- `"Failed to execute statement"` - Query execution failed
- `"Failed to listen"` - LISTEN command failed
- `"Failed to notify"` - NOTIFY command failed

### Connection Error Handling

The pool registers a no-op error handler to prevent unhandled rejection crashes:
```typescript
pool.on("error", (_err) => {})
```

Individual connections register error handlers that:
1. Release the connection back to the pool (with the error, so pool can destroy it)
2. Resume the pending effect with the connection error

### MigrationError

Used by `PgMigrator` when `pg_dump` fails:
```typescript
new Migrator.MigrationError({
  reason: "failed",
  message: error.message
})
```

### Timeout Handling

Connection timeout is enforced with `Effect.timeoutFail`:
```typescript
Effect.timeoutFail({
  duration: options.connectTimeout ?? Duration.seconds(5),
  onTimeout: () => new SqlError({
    cause: new Error("Connection timed out"),
    message: "PgClient: Connection timed out"
  })
})
```

### Cancellation as Error Recovery

When a query is interrupted (fiber interruption), `pg_cancel_backend()` is called best-effort. This allows the connection to be reused. The test suite verifies that after interrupting a `pg_sleep(1000)`, the connection still works for subsequent queries.

## Code Examples

Handling SQL errors:
```typescript
import { SqlError } from "@effect/sql/SqlError"

yield* sql`SELECT * FROM nonexistent`.pipe(
  Effect.catchTag("SqlError", (err) =>
    Console.error("SQL failed:", err.message)
  )
)
```

## Related Files

- `src/PgClient.ts` - All `SqlError` construction sites
- `src/PgMigrator.ts` - `MigrationError` usage
