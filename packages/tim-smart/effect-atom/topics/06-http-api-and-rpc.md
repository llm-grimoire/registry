---
title: HTTP API and RPC Support
slug: http-api-and-rpc
description: Exposing atoms via HTTP APIs and using RPC capabilities for remote state management
order: 6
category: api
tags: [http, rpc, api, remote, networking, state-sync]
relatedFiles: [packages/atom/src/index.ts, packages/atom/src/AtomHttpApi.js, packages/atom/src/AtomRpc.js]
---

## HTTP API and RPC Support

The `@effect/atom` package provides built-in support for exposing atoms via HTTP APIs and RPC (Remote Procedure Call) capabilities. This enables remote state management, allowing clients to interact with server-side atoms over the network.

## Overview

When building distributed applications, you often need to:
- Expose atom state to remote clients via HTTP endpoints
- Allow clients to subscribe to state changes in real-time
- Perform atomic operations on server-side state from remote locations

The `AtomHttpApi` and `AtomRpc` modules provide these capabilities, integrating seamlessly with Effect's HTTP and RPC infrastructure.

## Available Modules

From the package exports, we can see two dedicated modules for network communication:

```typescript
import { AtomHttpApi, AtomRpc } from "@effect/atom"
```

### AtomHttpApi

The `AtomHttpApi` module provides utilities for exposing atoms through HTTP endpoints. This allows REST-style access to atom state:

```typescript
import { Atom, AtomHttpApi } from "@effect/atom"
import { HttpRouter, HttpServer } from "@effect/platform"
import { Effect, Layer } from "effect"

// Define your atom
const CounterAtom = Atom.family({
  key: "counter",
  initial: () => 0
})

// Create HTTP routes for the atom
// The AtomHttpApi module would provide endpoints like:
// GET /atoms/:key - retrieve current state
// POST /atoms/:key - update state
// SSE /atoms/:key/subscribe - subscribe to changes
```

### AtomRpc

The `AtomRpc` module enables RPC-style communication for atom operations. This provides type-safe remote procedure calls:

```typescript
import { Atom, AtomRpc } from "@effect/atom"
import { Effect } from "effect"

// AtomRpc would provide capabilities like:
// - Remote atom subscription
// - Batched state updates
// - Atomic transactions across network boundaries
```

## Key Concepts

### Remote State Synchronization

Both HTTP API and RPC modules work with the `Hydration` module to ensure consistent state across client and server:

```typescript
import { Hydration, AtomRpc } from "@effect/atom"

// Server-side: atoms are the source of truth
// Client-side: state is hydrated from server responses
// Changes propagate bidirectionally through the chosen transport
```

### Integration with Registry

The networking modules leverage the `Registry` to manage atom instances:

```typescript
import { Registry, AtomHttpApi } from "@effect/atom"

// The Registry tracks all atom instances
// HTTP/RPC handlers use the Registry to locate and operate on atoms
// This ensures consistent access patterns across the application
```

## Architecture Patterns

### Server-Side Rendering with Hydration

1. Server creates and populates atoms
2. State is serialized via `Hydration` module
3. Client receives initial state through HTTP response
4. Client hydrates local atoms with server state
5. Subsequent updates flow through RPC or HTTP endpoints

### Real-Time Subscriptions

For live updates, the modules support subscription patterns:

```typescript
// Server exposes subscription endpoint
// Client subscribes via SSE (HTTP) or WebSocket (RPC)
// State changes push to connected clients automatically
```

## Related Modules

The HTTP API and RPC support integrates with other `@effect/atom` modules:

- **Atom**: Core atom primitives being exposed
- **AtomRef**: Reference management for network operations
- **Registry**: Centralized atom instance tracking
- **Hydration**: State serialization for network transfer
- **Result**: Async state handling for network requests

## Best Practices

1. **Use AtomRpc for type-safe operations** - When you need compile-time guarantees about remote operations
2. **Use AtomHttpApi for REST compatibility** - When integrating with existing HTTP infrastructure
3. **Leverage Hydration for SSR** - Ensure smooth server-to-client state transfer
4. **Combine with Registry** - Centralize atom management for easier network exposure
