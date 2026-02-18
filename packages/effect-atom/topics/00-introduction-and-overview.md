---
title: Introduction to Effect-Atom
slug: introduction-and-overview
description: Overview of the atom library, its purpose, core concepts, and how it integrates with Effect-TS for reactive state management
order: 0
category: guides
tags: [introduction, overview, state-management, reactive, effect-ts, atoms]
relatedFiles: [packages/atom/src/index.ts, README.md, packages/atom/README.md]
---

## Overview

Effect-Atom is a reactive state management library built on top of Effect-TS. It provides a powerful and type-safe way to manage application state with first-class support for effects, streams, and dependency injection through Effect's service layer.

This documentation introduces the core concepts and modules that make up the Effect-Atom library, helping you understand how the pieces fit together before diving into specific features.

## Why Effect-Atom?

Modern applications require sophisticated state management that goes beyond simple value storage. Effect-Atom addresses several key challenges:

- **Reactive State**: Atoms automatically track dependencies and propagate changes
- **Effect Integration**: Seamlessly work with async operations, errors, and resources
- **Type Safety**: Full TypeScript support with precise type inference
- **Resource Management**: Automatic cleanup with Effect's Scope system
- **Dependency Injection**: First-class support for Effect services and layers

## Core Modules

The library exports several key modules, each serving a specific purpose:

```typescript
// Main exports from @effect-atom/atom
export * as Atom from "./Atom.js"         // Core atom creation and manipulation
export * as AtomHttpApi from "./AtomHttpApi.js"  // HTTP API integration
export * as AtomRef from "./AtomRef.js"   // Atom reference management
export * as AtomRpc from "./AtomRpc.js"   // RPC integration
export * as Hydration from "./Hydration.js"  // Server-side hydration support
export * as Registry from "./Registry.js"    // Atom registry for tracking
export * as Result from "./Result.js"        // Result type for effectful atoms
```

### Atom

The `Atom` module is the heart of the library. Atoms are reactive state containers that can hold simple values, derived computations, or effectful operations.

```typescript
import { Atom } from "@effect-atom/atom-react"

// Simple value atom
const countAtom = Atom.make(0)

// Derived atom with dependency tracking
const doubleCountAtom = Atom.make((get) => get(countAtom) * 2)

// Alternative: using Atom.map for simple transformations
const tripleCountAtom = Atom.map(countAtom, (count) => count * 3)
```

### Result

When working with effectful atoms, values are wrapped in a `Result` type that handles loading states, errors, and success cases:

```typescript
import { Atom, Result } from "@effect-atom/atom-react"
import { Effect } from "effect"

// Effectful atom returns Result.Result<number>
const asyncCountAtom = Atom.make(Effect.succeed(0))

// Access results in other effectful atoms
const resultWithContextAtom = Atom.make(
  Effect.fnUntraced(function* (get: Atom.Context) {
    const count = yield* get.result(asyncCountAtom)
    return count + 1
  }),
)
```

### AtomRef

The `AtomRef` module provides reference management for atoms, allowing you to hold and manipulate atom instances.

### Registry

The `Registry` module tracks atom instances across your application, enabling features like debugging, persistence, and server-side rendering.

### Hydration

The `Hydration` module supports server-side rendering scenarios where atom state needs to be serialized on the server and rehydrated on the client.

### AtomHttpApi & AtomRpc

These modules provide integration with Effect's HTTP and RPC systems, allowing you to create atoms that interact with backend services.

## Installation

For React applications, install the React integration package which includes the core library:

```bash
pnpm add @effect-atom/atom-react
```

## Basic Usage Example

Here's a complete example showing the fundamental concepts:

```tsx
import { Atom, useAtomValue, useAtomSet } from "@effect-atom/atom-react"

// Create a persistent atom that survives component unmounts
const countAtom = Atom.make(0).pipe(
  Atom.keepAlive,
)

function App() {
  return (
    <div>
      <Counter />
      <CounterButton />
    </div>
  )
}

function Counter() {
  const count = useAtomValue(countAtom)
  return <h1>{count}</h1>
}

function CounterButton() {
  const setCount = useAtomSet(countAtom)
  return (
    <button onClick={() => setCount((count) => count + 1)}>Increment</button>
  )
}
```

## Key Concepts

### Atom Lifecycle

By default, atoms are reset when no longer used (no active subscribers). This automatic cleanup prevents memory leaks and ensures resources are properly released. Use `Atom.keepAlive` to persist atom values across component lifecycles.

### Dependency Tracking

When you use `get` inside an atom's computation function, Effect-Atom automatically tracks that dependency. When the source atom changes, derived atoms recompute automatically.

### Scoped Effects

All effectful atoms receive a `Scope`, allowing you to register finalizers that run when the atom is rebuilt or disposed:

```typescript
const resourceAtom = Atom.make(
  Effect.gen(function* () {
    yield* Effect.addFinalizer(() => Effect.log("cleanup"))
    return "resource"
  }),
)
```

### Service Integration

Create runtime atoms from Effect layers to inject services into your atoms:

```typescript
import { Effect } from "effect"

class Users extends Effect.Service<Users>()("app/Users", {
  effect: Effect.gen(function* () {
    const getAll = Effect.succeed([{ id: "1", name: "Alice" }])
    return { getAll } as const
  }),
}) {}

const runtimeAtom = Atom.runtime(Users.Default)
const usersAtom = runtimeAtom.atom(
  Effect.gen(function* () {
    const users = yield* Users
    return yield* users.getAll
  }),
)
```

## Next Steps

- Learn about [creating and managing atoms](/docs/atom-creation-and-basic-usage)
- Explore [derived state and computations](/docs/derived-atoms)
- Understand [effectful atoms and the Result type](/docs/effectful-atoms)
- Discover [service integration with AtomRuntime](/docs/atom-runtime-and-services)
