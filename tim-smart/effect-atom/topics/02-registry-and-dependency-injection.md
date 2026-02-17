---
title: Registry and Dependency Injection
slug: registry-and-dependency-injection
description: Understanding the Registry system for managing atom instances and enabling dependency injection across your application
order: 2
category: architecture
tags: [registry, dependency-injection, context, layer, state-management, effect]
relatedFiles: [packages/atom/src/Registry.ts, packages/atom/src/index.ts, packages/atom/src/Atom.js]
---

## Overview

The Registry is the central state container in effect-atom that manages all atom instances and their values. It provides a unified system for creating, reading, writing, and subscribing to atoms while integrating seamlessly with Effect's dependency injection system through Context and Layers.

## What is a Registry?

A Registry is an object that:
- Stores the current values of all atoms
- Manages atom dependencies and recomputation
- Handles subscriptions for reactive updates
- Provides lifecycle management (mounting, disposing)
- Integrates with Effect's Context for dependency injection

## Creating a Registry

### Basic Construction

The simplest way to create a registry is with `Registry.make`:

```typescript
import * as Registry from "@effect/atom/Registry"

const registry = Registry.make()
```

### With Configuration Options

You can customize registry behavior with options:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

const counterAtom = Atom.make(0)

const registry = Registry.make({
  // Pre-populate atoms with initial values
  initialValues: [[counterAtom, 10]],
  
  // Custom task scheduler for batching updates
  scheduleTask: (f) => queueMicrotask(f),
  
  // Timeout resolution for debouncing (in ms)
  timeoutResolution: 100,
  
  // Default time-to-live for idle atoms (in ms)
  defaultIdleTTL: 5000
})
```

## Registry Interface

The Registry provides these core methods:

```typescript
export interface Registry {
  // Read the current value of an atom
  readonly get: <A>(atom: Atom.Atom<A>) => A
  
  // Set a new value on a writable atom
  readonly set: <R, W>(atom: Atom.Writable<R, W>, value: W) => void
  
  // Update an atom using a function
  readonly update: <R, W>(atom: Atom.Writable<R, W>, f: (_: R) => W) => void
  
  // Modify with return value (like getAndSet)
  readonly modify: <R, W, A>(
    atom: Atom.Writable<R, W>, 
    f: (_: R) => [returnValue: A, nextValue: W]
  ) => A
  
  // Subscribe to value changes
  readonly subscribe: <A>(
    atom: Atom.Atom<A>, 
    f: (_: A) => void, 
    options?: { readonly immediate?: boolean }
  ) => () => void
  
  // Mount an atom (keeps it alive)
  readonly mount: <A>(atom: Atom.Atom<A>) => () => void
  
  // Force refresh a derived atom
  readonly refresh: <A>(atom: Atom.Atom<A>) => void
  
  // Reset all atoms to initial state
  readonly reset: () => void
  
  // Clean up all resources
  readonly dispose: () => void
}
```

## Dependency Injection with Effect

### The AtomRegistry Tag

effect-atom provides an `AtomRegistry` Context Tag for dependency injection:

```typescript
import * as Context from "effect/Context"
import { Registry } from "@effect/atom"

// AtomRegistry is defined as:
export class AtomRegistry extends Context.Tag("@effect/atom/Registry/CurrentRegistry")<
  AtomRegistry,
  Registry
>() {}
```

### Using Layers

Create a Layer to provide the registry to your application:

```typescript
import * as Effect from "effect/Effect"
import * as Layer from "effect/Layer"
import { Registry, Atom } from "@effect/atom"

// Basic layer with default options
const registryLayer: Layer.Layer<Registry.AtomRegistry> = Registry.layer

// Layer with custom options
const customRegistryLayer = Registry.layerOptions({
  defaultIdleTTL: 10000,
  timeoutResolution: 50
})

// Using the registry in an Effect program
const program = Effect.gen(function* () {
  const registry = yield* Registry.AtomRegistry
  
  const counterAtom = Atom.make(0)
  
  // Read value
  const count = registry.get(counterAtom)
  
  // Update value
  registry.set(counterAtom, count + 1)
  
  return registry.get(counterAtom)
})

// Provide the layer and run
const result = await Effect.runPromise(
  program.pipe(Effect.provide(Registry.layer))
)
```

### Scoped Registry Lifecycle

The `layerOptions` function creates a scoped layer that automatically disposes the registry when the scope closes:

```typescript
export const layerOptions = (options?: {
  readonly initialValues?: Iterable<readonly [Atom.Atom<any>, any]> | undefined
  readonly scheduleTask?: ((f: () => void) => void) | undefined
  readonly timeoutResolution?: number | undefined
  readonly defaultIdleTTL?: number | undefined
}): Layer.Layer<AtomRegistry> =>
  Layer.scoped(
    AtomRegistry,
    Effect.gen(function*() {
      const scope = yield* Effect.scope
      const scheduler = yield* FiberRef.get(FiberRef.currentScheduler)
      const registry = internal.make({
        ...options,
        scheduleTask: options?.scheduleTask ?? ((f) => scheduler.scheduleTask(f, 0))
      })
      // Automatically dispose when scope closes
      yield* Scope.addFinalizer(scope, Effect.sync(() => registry.dispose()))
      return registry
    })
  )
```

## Converting Atoms to Streams

The Registry provides utilities to convert atom subscriptions into Effect Streams:

### Basic Stream Conversion

```typescript
import * as Stream from "effect/Stream"
import { Registry, Atom } from "@effect/atom"

const counterAtom = Atom.make(0)

// Convert atom to a Stream of values
const counterStream: Stream.Stream<number> = Registry.toStream(registry, counterAtom)

// Or using the curried form
const toCounterStream = Registry.toStream(counterAtom)
const stream = toCounterStream(registry)
```

### Result Stream Conversion

For atoms containing `Result` values, use `toStreamResult` to get a typed error channel:

```typescript
import { Registry, Atom, Result } from "@effect/atom"
import * as Stream from "effect/Stream"

const asyncAtom = Atom.make<Result.Result<string, Error>>(Result.initial())

// Stream with proper error handling
const resultStream: Stream.Stream<string, Error> = Registry.toStreamResult(
  registry, 
  asyncAtom
)
```

The `toStreamResult` function:
1. Filters out initial (loading) states
2. Maps success values to the success channel
3. Maps failures to the error channel with proper cause handling

```typescript
export const toStreamResult = dual(
  2,
  <A, E>(self: Registry, atom: Atom.Atom<Result.Result<A, E>>): Stream.Stream<A, E> =>
    toStream(self, atom).pipe(
      Stream.filter(Result.isNotInitial),
      Stream.mapEffect((result) =>
        result._tag === "Success" 
          ? Effect.succeed(result.value) 
          : Effect.failCause(result.cause)
      )
    )
)
```

## Type Guards

Check if a value is a Registry:

```typescript
import { Registry } from "@effect/atom"

const maybeRegistry: unknown = getRegistry()

if (Registry.isRegistry(maybeRegistry)) {
  // TypeScript knows this is a Registry
  const value = maybeRegistry.get(someAtom)
}
```

## Best Practices

1. **Single Registry Per Application**: Use one registry at the root of your application to ensure consistent state

2. **Use Layers for Testing**: Create test layers with pre-populated initial values:
   ```typescript
   const testLayer = Registry.layerOptions({
     initialValues: [[userAtom, mockUser], [configAtom, testConfig]]
   })
   ```

3. **Clean Up Resources**: When not using Layers, always call `dispose()` when done:
   ```typescript
   const registry = Registry.make()
   try {
     // ... use registry
   } finally {
     registry.dispose()
   }
   ```

4. **Mount Long-Lived Atoms**: Use `mount` for atoms that should stay active:
   ```typescript
   const unmount = registry.mount(sessionAtom)
   // Later when done
   unmount()
   ```
