---
title: Observability and Span Attributes
slug: observability
description: OpenTelemetry span attributes and tracing integration
order: 12
category: patterns
tags: [tracing, spans, opentelemetry, observability]
relatedFiles: [src/PgClient.ts]
---

# Observability and Span Attributes

## Overview

`@effect/sql-pg` integrates with Effect's tracing system by attaching semantic span attributes to SQL operations. These follow OpenTelemetry database conventions.

## Key Concepts

### Default Span Attributes

Every `PgClient` instance automatically includes these span attributes on SQL operations:

```typescript
[ATTR_DB_SYSTEM_NAME, "postgresql"],
[ATTR_DB_NAMESPACE, config.database ?? config.username ?? "postgres"],
[ATTR_SERVER_ADDRESS, config.host ?? "localhost"],
[ATTR_SERVER_PORT, config.port ?? 5432]
```

The attribute keys follow OTel semantic conventions:
- `db.system.name` - Always `"postgresql"`
- `db.namespace` - Database name (falls back to username, then `"postgres"`)
- `server.address` - Host (defaults to `"localhost"`)
- `server.port` - Port (defaults to `5432`)

### Custom Span Attributes

Additional span attributes can be provided via config:

```typescript
PgClient.layer({
  database: "mydb",
  spanAttributes: {
    "service.name": "my-api",
    "deployment.environment": "production"
  }
})
```

Custom attributes are prepended to the default ones:
```typescript
spanAttributes: [
  ...Object.entries(options.spanAttributes),  // custom first
  [ATTR_DB_SYSTEM_NAME, "postgresql"],        // then defaults
  ...
]
```

### Integration with Effect Tracing

These attributes are passed to `Client.make()` from `@effect/sql`, which uses them to create spans around SQL operations. The spans integrate with Effect's tracing infrastructure and can be exported to any OpenTelemetry-compatible backend.

The resolver example shows integration with `@effect/experimental/DevTools`:
```typescript
import * as DevTools from "@effect/experimental/DevTools"

program.pipe(
  Effect.provide(PgLive.pipe(
    Layer.provide(DevTools.layer())
  ))
)
```

## Related Files

- `src/PgClient.ts` - Span attribute constants and wiring in `makeClient`
- `examples/resolver.ts` - DevTools integration example
