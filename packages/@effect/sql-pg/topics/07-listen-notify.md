---
title: LISTEN/NOTIFY
slug: listen-notify
description: PostgreSQL pub/sub via sql.listen() and sql.notify()
order: 7
category: api
tags: [listen, notify, pubsub, stream, realtime]
relatedFiles: [src/PgClient.ts, examples/listen-notify.ts]
---

# LISTEN/NOTIFY

## Overview

`PgClient` exposes PostgreSQL's LISTEN/NOTIFY mechanism as Effect-native APIs. `listen()` returns a `Stream` of notification payloads, and `notify()` sends a notification to a channel.

## Key Concepts

### listen(channel)

Returns `Stream.Stream<string, SqlError>`. Internally:

1. Acquires a dedicated connection via `RcRef` (not from the regular pool — this connection stays open)
2. Sends `LISTEN "channel"` to PostgreSQL
3. Registers a `notification` event handler on the `pg` client
4. Emits `msg.payload` for matching channel notifications
5. On stream finalization: sends `UNLISTEN "channel"`, removes event handler

Channel names are escaped with `Pg.escapeIdentifier()` to prevent injection.

```typescript
listen: (channel: string) =>
  Stream.asyncPush<string, SqlError>(Effect.fnUntraced(function*(emit) {
    const client = yield* RcRef.get(listenClient)
    function onNotification(msg: Pg.Notification) {
      if (msg.channel === channel && msg.payload) {
        emit.single(msg.payload)
      }
    }
    yield* Effect.addFinalizer(() =>
      Effect.promise(() => {
        client.off("notification", onNotification)
        return client.query(`UNLISTEN ${Pg.escapeIdentifier(channel)}`)
      })
    )
    yield* Effect.tryPromise({
      try: () => client.query(`LISTEN ${Pg.escapeIdentifier(channel)}`),
      catch: (cause) => new SqlError({ cause, message: "Failed to listen" })
    })
    client.on("notification", onNotification)
  }))
```

### notify(channel, payload)

Sends `NOTIFY "channel", $1` using a pooled connection (not the dedicated listener connection):

```typescript
notify: (channel: string, payload: string) =>
  Effect.async<void, SqlError>((resume) => {
    pool.query(`NOTIFY ${Pg.escapeIdentifier(channel)}, $1`, [payload], (err) => {
      if (err) resume(Effect.fail(new SqlError({...})))
      else resume(Effect.void)
    })
  })
```

### Dedicated Listener Connection

The listener uses an `RcRef` for its connection, separate from the pool. This is important because:
- LISTEN requires a persistent connection (notifications are connection-specific)
- The connection is lazily acquired and reference-counted
- Multiple `listen()` streams can share the same underlying connection

## Code Examples

From `examples/listen-notify.ts`:
```typescript
const program = Effect.gen(function*() {
  const sql = yield* PgClient.PgClient

  // Fork listener as background stream
  yield* sql.listen("channel_name").pipe(
    Stream.tap((message) => Console.log("Received message", message)),
    Stream.runDrain,
    Effect.forkScoped
  )

  // Send notifications
  yield* sql.notify("channel_name", "Hello, world!").pipe(
    Effect.tap(() => Effect.sleep("1 second")),
    Effect.replicateEffect(5)
  )
}).pipe(Effect.scoped)

program.pipe(
  Effect.provide(PgClient.layer({
    database: "postgres",
    username: "postgres"
  })),
  Effect.runFork
)
```

## Related Files

- `src/PgClient.ts` - `listen` and `notify` implementations, `listenClient` RcRef
- `examples/listen-notify.ts` - Complete working example
