---
title: SolidJS Integration Guide
slug: solidjs-integration
description: Using atoms with SolidJS applications through dedicated hooks and reactive bindings
order: 3
category: guides
tags: [solidjs, integration, hooks, reactive, ui, frontend]
relatedFiles: [packages/atom-solid/src/index.ts, packages/atom-solid/src/Hooks.ts, packages/atom-solid/src/RegistryContext.ts]
---

## Overview

The `@effect-atom/solid` package provides seamless integration between Effect Atom and SolidJS applications. It offers reactive hooks and context providers that allow you to use atoms within SolidJS components while maintaining full reactivity and type safety.

SolidJS's fine-grained reactivity model pairs exceptionally well with Effect Atom's state management approach, enabling efficient updates and minimal re-renders.

## Package Exports

The package re-exports all core atom functionality along with SolidJS-specific utilities:

```typescript
import {
  Atom,
  Registry,
  Result,
  AtomRef,
  AtomHttpApi,
  AtomRpc,
  Hydration
} from "@effect-atom/solid"
```

This allows you to use a single import for both atom core functionality and SolidJS integration.

## Key Concepts

### Registry Context

The Registry Context provides a way to make an atom registry available throughout your SolidJS component tree. This follows SolidJS's context pattern for dependency injection.

```typescript
import { RegistryContext } from "@effect-atom/solid"
import { Registry } from "@effect-atom/solid"

// Create your registry
const registry = Registry.make()

// Provide it to your app
function App() {
  return (
    <RegistryContext.Provider value={registry}>
      <YourComponents />
    </RegistryContext.Provider>
  )
}
```

### Reactive Hooks

The hooks exported from the package enable reactive consumption of atom values within SolidJS components. These hooks integrate with SolidJS's reactivity system to ensure your components update efficiently when atom values change.

```typescript
import { useAtom, useAtomValue } from "@effect-atom/solid"
import { Atom } from "@effect-atom/solid"

// Define an atom
const counterAtom = Atom.make("counter", () => 0)

function Counter() {
  // Get reactive value and setter
  const [count, setCount] = useAtom(counterAtom)
  
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count()}
    </button>
  )
}

function CounterDisplay() {
  // Read-only reactive value
  const count = useAtomValue(counterAtom)
  
  return <span>Current count: {count()}</span>
}
```

## Working with Async Atoms

The Result type helps manage loading states and errors when working with async atoms:

```typescript
import { useAtomValue, Result, Atom } from "@effect-atom/solid"
import { Effect } from "effect"

const userAtom = Atom.makeAsync("user", () => 
  Effect.promise(() => fetch("/api/user").then(r => r.json()))
)

function UserProfile() {
  const userResult = useAtomValue(userAtom)
  
  return (
    <Switch>
      <Match when={Result.isLoading(userResult())}>
        <LoadingSpinner />
      </Match>
      <Match when={Result.isSuccess(userResult())}>
        <UserCard user={Result.getValue(userResult())} />
      </Match>
      <Match when={Result.isError(userResult())}>
        <ErrorMessage error={Result.getError(userResult())} />
      </Match>
    </Switch>
  )
}
```

## Server-Side Rendering & Hydration

The Hydration module supports SSR scenarios common in SolidStart applications:

```typescript
import { Hydration, Registry } from "@effect-atom/solid"

// On the server: serialize atom state
const serializedState = Hydration.serialize(registry)

// On the client: hydrate from server state
const registry = Registry.make()
Hydration.hydrate(registry, serializedState)
```

## API Integration

For data fetching, you can use either AtomHttpApi for standard HTTP endpoints or AtomRpc for Effect RPC integration:

```typescript
import { AtomHttpApi, AtomRpc } from "@effect-atom/solid"

// These modules provide utilities for creating atoms
// that fetch data from your backend services
```

## Best Practices

1. **Single Registry**: Use one registry per application, provided via context
2. **Atom Colocation**: Define atoms close to where they're used, or in dedicated modules for shared state
3. **Leverage Fine-Grained Reactivity**: SolidJS will only re-render what changes, so structure your atoms accordingly
4. **Use AtomRef for Derived State**: Create computed atoms that derive from other atoms for efficient updates
