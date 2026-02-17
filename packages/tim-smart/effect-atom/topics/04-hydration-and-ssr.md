---
title: Hydration and Server-Side Rendering
slug: hydration-and-ssr
description: Handling state hydration for SSR scenarios and ensuring consistent state between server and client
order: 4
category: patterns
tags: [ssr, hydration, server-side-rendering, serialization, state-transfer]
relatedFiles: [packages/atom/src/Hydration.ts, packages/atom/src/Atom.ts, packages/atom/src/Registry.ts, packages/atom/src/Result.ts]
---

## Overview

Server-Side Rendering (SSR) requires transferring state from the server to the client so the application can "hydrate" with the correct initial state without refetching data. Effect-Atom provides a complete hydration system that handles serialization, deserialization, and async state resolution.

This guide covers:
- Dehydrating state on the server for transfer to the client
- Hydrating state on the client from server-rendered data
- Handling async atoms that are still loading during SSR
- Working with `Result` types in SSR scenarios

## Key Concepts

### Dehydration

Dehydration is the process of serializing atom state on the server so it can be embedded in HTML and sent to the client. Only atoms marked as serializable (using `Atom.serializable`) will be included.

### Hydration

Hydration is the reverse process—taking the serialized state from the server and restoring it in the client's registry before the application renders.

### DehydratedAtom

The `DehydratedAtom` interface represents a serialized atom value:

```typescript
interface DehydratedAtomValue extends DehydratedAtom {
  readonly key: string
  readonly value: unknown
  readonly dehydratedAt: number
  readonly resultPromise?: Promise<unknown> | undefined
}
```

- `key`: The unique identifier for the atom in the registry
- `value`: The serialized value
- `dehydratedAt`: Timestamp when dehydration occurred
- `resultPromise`: Optional promise for handling async atoms in Initial state

## Server-Side Dehydration

Use `dehydrate` to extract all serializable atom values from a registry:

```typescript
import * as Hydration from "@effect-atom/atom/Hydration"
import * as Registry from "@effect-atom/atom/Registry"

// After rendering on the server
const registry = Registry.make()

// ... atoms are populated during render ...

// Dehydrate state for transfer to client
const dehydratedState = Hydration.dehydrate(registry)

// Embed in HTML
const html = `
  <script>
    window.__ATOM_STATE__ = ${JSON.stringify(dehydratedState)}
  </script>
`
```

### Handling Initial States

The `encodeInitialAs` option controls how `Result.Initial` values are handled during dehydration:

```typescript
// Option 1: Ignore atoms in Initial state (default)
const state1 = Hydration.dehydrate(registry, {
  encodeInitialAs: "ignore"
})

// Option 2: Include only the value, treating Initial as a regular value
const state2 = Hydration.dehydrate(registry, {
  encodeInitialAs: "value-only"
})

// Option 3: Create a promise that resolves when the atom loads
const state3 = Hydration.dehydrate(registry, {
  encodeInitialAs: "promise"
})
```

The `promise` mode is particularly powerful for streaming SSR scenarios where you want to send initial HTML immediately but resolve async data as it becomes available.

## Client-Side Hydration

Use `hydrate` to restore state from the server:

```typescript
import * as Hydration from "@effect-atom/atom/Hydration"
import * as Registry from "@effect-atom/atom/Registry"

// Create client registry
const registry = Registry.make()

// Hydrate from server state (before rendering)
if (typeof window !== "undefined" && window.__ATOM_STATE__) {
  Hydration.hydrate(registry, window.__ATOM_STATE__)
}

// Now render the application with pre-populated state
```

### Async Hydration with Promises

When atoms were dehydrated with `encodeInitialAs: "promise"`, the hydration process automatically handles promise resolution:

```typescript
// The hydrate function automatically processes resultPromise values
Hydration.hydrate(registry, dehydratedState)

// If a dehydrated atom had a resultPromise, hydrate will:
// 1. Set the initial value immediately
// 2. Wait for the promise to resolve
// 3. Update the atom with the resolved value
```

This happens transparently in the `hydrate` implementation:

```typescript
// From Hydration.ts - promise resolution logic
if (!datom.resultPromise) continue
datom.resultPromise.then((resolvedValue) => {
  const nodes = (registry as any).getNodes()
  const node = nodes.get(datom.key)
  if (node) {
    const atom = node.atom as any
    if (atom[Atom.SerializableTypeId]) {
      const decoded = atom[Atom.SerializableTypeId].decode(resolvedValue)
      node.setValue(decoded)
    }
  } else {
    registry.setSerializable(datom.key, resolvedValue)
  }
})
```

## Working with DehydratedAtom Arrays

Use `toValues` to access the underlying array with full type information:

```typescript
import * as Hydration from "@effect-atom/atom/Hydration"

const dehydrated = Hydration.dehydrate(registry)
const values = Hydration.toValues(dehydrated)

// Now you can inspect individual atoms
for (const atom of values) {
  console.log(`Atom ${atom.key} dehydrated at ${atom.dehydratedAt}`)
  console.log(`Value:`, atom.value)
}
```

## Complete SSR Example

### Server (e.g., Next.js getServerSideProps)

```typescript
import * as Hydration from "@effect-atom/atom/Hydration"
import * as Registry from "@effect-atom/atom/Registry"

export async function getServerSideProps() {
  const registry = Registry.make()
  
  // Pre-populate atoms with data
  // ... fetch data and set atom values ...
  
  const dehydratedState = Hydration.dehydrate(registry, {
    encodeInitialAs: "value-only"
  })
  
  return {
    props: {
      atomState: dehydratedState
    }
  }
}
```

### Client (App component)

```typescript
import * as Hydration from "@effect-atom/atom/Hydration"
import { useRegistry } from "@effect-atom/atom-react"
import { useEffect, useRef } from "react"

function App({ atomState }) {
  const registry = useRegistry()
  const hydrated = useRef(false)
  
  // Hydrate only once on mount
  if (!hydrated.current && atomState) {
    Hydration.hydrate(registry, atomState)
    hydrated.current = true
  }
  
  return <YourApp />
}
```

## Requirements for SSR

1. **Serializable Atoms**: Only atoms created with `Atom.serializable` or `AtomFamily.serializable` will be included in dehydration
2. **Stable Keys**: Atom keys must be identical between server and client
3. **Hydration Timing**: Always hydrate before your first render to avoid hydration mismatches
4. **JSON-Compatible Values**: All serialized values must be JSON-compatible
