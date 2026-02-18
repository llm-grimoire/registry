---
title: Client Configuration
slug: configuration
description: PgClientConfig options for connection, pooling, transforms, and SSL
order: 2
category: setup
tags: [config, connection, pool, ssl, transforms]
relatedFiles: [src/PgClient.ts, test/utils.ts]
---

# Client Configuration

## Overview

`PgClientConfig` defines all options for creating a `PgClient`. It covers connection details, pool tuning, name transforms, and custom type handling. Config can be provided directly or parsed from a connection URL.

## Key Concepts

### PgClientConfig Interface

```typescript
export interface PgClientConfig {
  // Connection
  readonly url?: Redacted.Redacted | undefined
  readonly host?: string | undefined
  readonly port?: number | undefined
  readonly path?: string | undefined
  readonly ssl?: boolean | ConnectionOptions | undefined
  readonly database?: string | undefined
  readonly username?: string | undefined
  readonly password?: Redacted.Redacted | undefined

  // Custom stream (e.g., for SSH tunneling)
  readonly stream?: (() => Duplex) | undefined

  // Pool tuning
  readonly idleTimeout?: Duration.DurationInput | undefined
  readonly connectTimeout?: Duration.DurationInput | undefined
  readonly maxConnections?: number | undefined
  readonly minConnections?: number | undefined
  readonly connectionTTL?: Duration.DurationInput | undefined

  // Metadata
  readonly applicationName?: string | undefined
  readonly spanAttributes?: Record<string, unknown> | undefined

  // Transforms
  readonly transformResultNames?: ((str: string) => string) | undefined
  readonly transformQueryNames?: ((str: string) => string) | undefined
  readonly transformJson?: boolean | undefined
  readonly types?: Pg.CustomTypesConfig | undefined
}
```

### URL vs Individual Fields

When `url` is provided, the pool connects via connection string. Individual fields (`host`, `port`, `database`, etc.) are still checked and merged -- explicit fields take priority over URL-parsed values. After connection, `config` on the client reflects the merged result.

Passwords and URLs use `Redacted.Redacted` to prevent accidental logging:
```typescript
PgClient.layer({
  url: Redacted.make("postgresql://user:pass@host:5432/db")
})
```

### Connection Pool Settings

- `maxConnections` / `minConnections` - Pool size bounds (maps to `pg.Pool` `max`/`min`)
- `idleTimeout` - How long idle connections stay in pool (maps to `idleTimeoutMillis`)
- `connectTimeout` - Timeout for initial connection (default: 5 seconds). Also used as timeout for the startup `SELECT 1` health check.
- `connectionTTL` - Maximum lifetime of a connection (maps to `maxLifetimeSeconds`)
- `applicationName` - Defaults to `"@effect/sql-pg"`

### Name Transforms

Two transform functions enable automatic camelCase/snake_case conversion:
- `transformQueryNames` - Applied to identifiers in SQL (column names in INSERT/UPDATE). `camelToSnake` means you write camelCase in TS, it becomes snake_case in SQL.
- `transformResultNames` - Applied to result column names. `snakeToCamel` means DB snake_case columns come back as camelCase in TS.

```typescript
import { String } from "effect"

PgClient.layer({
  database: "mydb",
  transformQueryNames: String.camelToSnake,
  transformResultNames: String.snakeToCamel
})
```

The `transformJson` option (default `true`) controls whether transforms are applied recursively into JSON/array values in results.

## Code Examples

Minimal local connection:
```typescript
const PgLive = PgClient.layer({
  database: "postgres",
  username: "postgres"
})
```

Full production config:
```typescript
const PgLive = PgClient.layer({
  url: Redacted.make(process.env.DATABASE_URL!),
  ssl: true,
  maxConnections: 20,
  minConnections: 5,
  idleTimeout: "30 seconds",
  connectTimeout: "10 seconds",
  connectionTTL: "30 minutes",
  transformQueryNames: String.camelToSnake,
  transformResultNames: String.snakeToCamel
})
```

Config from Effect Config system:
```typescript
const PgLive = PgClient.layerConfig(
  Config.all({
    url: Config.redacted("DATABASE_URL"),
    maxConnections: Config.integer("PG_MAX_CONNECTIONS").pipe(
      Config.withDefault(10)
    )
  })
)
```

## Related Files

- `src/PgClient.ts` - `PgClientConfig` interface and `make` constructor
- `test/utils.ts` - Test layer examples showing URL-based and transform config
