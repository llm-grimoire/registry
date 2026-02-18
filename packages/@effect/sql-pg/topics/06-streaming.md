---
title: Streaming Queries
slug: streaming
description: Cursor-based streaming via pg-cursor and Effect Stream
order: 6
category: api
tags: [stream, cursor, pg-cursor, chunked]
relatedFiles: [src/PgClient.ts, test/Client.test.ts]
---

# Streaming Queries

## Overview

`@effect/sql-pg` supports streaming query results using `pg-cursor`, exposing them as `Effect.Stream`. This avoids loading entire result sets into memory for large queries.

## Key Concepts

### How It Works

The `.stream` property on any SQL statement returns a `Stream.Stream<Row, SqlError>`:

```typescript
const sql = yield* SqlClient.SqlClient
const rows = yield* sql`SELECT generate_series(1, 3)`.stream.pipe(
  Stream.runCollect,
  Effect.map(Chunk.toReadonlyArray)
)
// [{ generate_series: 1 }, { generate_series: 2 }, { generate_series: 3 }]
```

### Internal Implementation

Streaming is implemented in `ConnectionImpl.executeStream`:

1. A `Scope` is acquired
2. A `Pg.PoolClient` is reserved for the duration (or the pre-assigned client is used for transactions)
3. A `pg-cursor` (`Cursor`) is created with the query
4. A scope finalizer closes the cursor
5. Results are pulled in chunks of 128 rows via `cursor.read(128, ...)`
6. `Stream.repeatEffectChunkOption` drives the pull loop until the cursor is exhausted

```typescript
executeStream(sql, params, transformRows) {
  return Effect.gen(function*() {
    const cursor = client.query(new Cursor(sql, params))
    yield* Scope.addFinalizer(scope, Effect.promise(() => cursor.close()))
    const pull = Effect.async<Chunk.Chunk<any>, Option.Option<SqlError>>((resume) => {
      cursor.read(128, (err, rows) => {
        if (err) resume(Effect.fail(Option.some(new SqlError({...}))))
        else if (Arr.isNonEmptyArray(rows)) resume(Effect.succeed(Chunk.unsafeFromArray(rows)))
        else resume(Effect.fail(Option.none()))  // stream complete
      })
    })
    return Stream.repeatEffectChunkOption(pull)
  }).pipe(Stream.unwrapScoped)
}
```

### Chunk Size

The cursor reads 128 rows at a time. This is hardcoded and provides a good balance between memory usage and round-trip overhead.

### Transform Support

If `transformResultNames` is configured, the transform is applied to each chunk of rows before they're emitted into the stream.

## Code Examples

Streaming with processing:
```typescript
const sql = yield* SqlClient.SqlClient

yield* sql`SELECT * FROM large_table`.stream.pipe(
  Stream.tap((row) => processRow(row)),
  Stream.runDrain
)
```

Collecting stream results:
```typescript
const allRows = yield* sql`SELECT * FROM people`.stream.pipe(
  Stream.runCollect,
  Effect.map(Chunk.toReadonlyArray)
)
```

## Related Files

- `src/PgClient.ts` - `executeStream` method in `ConnectionImpl`
- `test/Client.test.ts` - Stream test using `generate_series`
