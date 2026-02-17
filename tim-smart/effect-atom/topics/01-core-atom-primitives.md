---
title: Core Atom Primitives
slug: core-atom-primitives
description: Deep dive into the fundamental building blocks: Atom, AtomRef, and how they provide reactive state containers
order: 1
category: architecture
tags: [atom, atomref, reactive, state-management, primitives, core]
relatedFiles: [packages/atom/src/Atom.ts, packages/atom/src/AtomRef.ts, packages/atom/src/index.ts]
---

## Overview

The `effect-atom` library provides two fundamental primitives for reactive state management: `Atom` and `AtomRef`. These building blocks enable fine-grained reactivity with automatic dependency tracking, making it easy to build complex reactive applications that integrate seamlessly with the Effect ecosystem.

## Atom: The Core Reactive Primitive

An `Atom` is a reactive container that holds a value and automatically tracks dependencies. When you read from an atom within another atom's computation, a dependency is established. This enables automatic re-computation when dependencies change.

### Atom Interface

```typescript
export interface Atom<A> extends Pipeable, Inspectable.Inspectable {
  readonly [TypeId]: TypeId
  readonly keepAlive: boolean
  readonly lazy: boolean
  readonly read: (get: Context) => A
  readonly refresh?: (f: <A>(atom: Atom<A>) => void) => void
  readonly label?: readonly [name: string, stack: string]
  readonly idleTTL?: number
}
```

### Key Properties

- **`keepAlive`**: When `true`, the atom's state persists even when no subscribers are active
- **`lazy`**: Controls whether the atom computes its value eagerly or lazily
- **`read`**: The function that computes the atom's value, receiving a `Context` for reading other atoms
- **`refresh`**: Optional function to trigger a refresh of dependencies
- **`idleTTL`**: Time-to-live in milliseconds for idle atoms before cleanup

### Type Guards and Utilities

```typescript
// Check if a value is an Atom
export const isAtom = (u: unknown): u is Atom<any> => hasProperty(u, TypeId)

// Extract types from atoms
export type Type<T extends Atom<any>> = T extends Atom<infer A> ? A : never
export type Success<T extends Atom<any>> = T extends Atom<Result.Result<infer A, infer _>> ? A : never
export type Failure<T extends Atom<any>> = T extends Atom<Result.Result<infer _, infer E>> ? E : never
```

## Writable Atoms

A `Writable` atom extends `Atom` with the ability to be updated. It has separate read and write types, enabling transformations during writes.

```typescript
export interface Writable<R, W = R> extends Atom<R> {
  readonly [WritableTypeId]: WritableTypeId
  readonly write: (ctx: WriteContext<R>, value: W) => void
}
```

The dual type parameters `R` (read) and `W` (write) allow for powerful patterns where the written value differs from the stored value.

## The Context API

The `Context` interface provides the primary way to interact with atoms during computation:

```typescript
export interface Context {
  // Basic reading
  <A>(atom: Atom<A>): A
  get<A>(this: Context, atom: Atom<A>): A
  once<A>(this: Context, atom: Atom<A>): A  // Read without establishing dependency

  // Result handling (for async atoms)
  result<A, E>(this: Context, atom: Atom<Result.Result<A, E>>, options?: {
    readonly suspendOnWaiting?: boolean | undefined
  }): Effect.Effect<A, E>
  resultOnce<A, E>(this: Context, atom: Atom<Result.Result<A, E>>, options?: {
    readonly suspendOnWaiting?: boolean | undefined
  }): Effect.Effect<A, E>

  // Writing
  set<R, W>(this: Context, atom: Writable<R, W>, value: W): void
  setResult<A, E, W>(this: Context, atom: Writable<Result.Result<A, E>, W>, value: W): Effect.Effect<A, E>
  setSelf<A>(this: Context, a: A): void

  // Lifecycle
  addFinalizer(this: Context, f: () => void): void
  mount<A>(this: Context, atom: Atom<A>): void
  refresh<A>(this: Context, atom: Atom<A>): void
  refreshSelf(this: Context): void

  // Streaming
  stream<A>(this: Context, atom: Atom<A>, options?: {
    readonly withoutInitialValue?: boolean
    readonly bufferSize?: number
  }): Stream.Stream<A>
  streamResult<A, E>(this: Context, atom: Atom<Result.Result<A, E>>, options?: {
    readonly withoutInitialValue?: boolean
    readonly bufferSize?: number
  }): Stream.Stream<A, E>

  // Subscriptions
  subscribe<A>(this: Context, atom: Atom<A>, f: (_: A) => void, options?: {
    readonly immediate?: boolean
  }): void

  readonly registry: Registry.Registry
}
```

## AtomRef: Fine-Grained Mutable References

`AtomRef` provides a different approach to reactivity - mutable references with subscription support. Unlike `Atom` which uses pull-based reactivity, `AtomRef` uses a push-based model.

### ReadonlyRef Interface

```typescript
export interface ReadonlyRef<A> extends Equal.Equal {
  readonly [TypeId]: TypeId
  readonly key: string
  readonly value: A
  readonly subscribe: (f: (a: A) => void) => () => void
  readonly map: <B>(f: (a: A) => B) => ReadonlyRef<B>
}
```

### AtomRef Interface

```typescript
export interface AtomRef<A> extends ReadonlyRef<A> {
  readonly prop: <K extends keyof A>(prop: K) => AtomRef<A[K]>
  readonly set: (value: A) => AtomRef<A>
  readonly update: (f: (value: A) => A) => AtomRef<A>
}
```

### Creating AtomRefs

```typescript
import { AtomRef } from "@effect-atom/atom"

// Create a simple ref
const counterRef = AtomRef.make(0)

// Access and modify
console.log(counterRef.value) // 0
counterRef.set(5)
counterRef.update(n => n + 1)
```

### Property Access

One of the most powerful features is the ability to create refs that focus on nested properties:

```typescript
interface User {
  name: string
  address: {
    city: string
    zip: string
  }
}

const userRef = AtomRef.make<User>({
  name: "Alice",
  address: { city: "NYC", zip: "10001" }
})

// Create a focused ref for the city
const cityRef = userRef.prop("address").prop("city")

cityRef.set("Boston") // Updates the parent automatically
```

### Subscriptions

```typescript
const ref = AtomRef.make(0)

// Subscribe to changes
const unsubscribe = ref.subscribe((value) => {
  console.log("Value changed:", value)
})

ref.set(1) // Logs: "Value changed: 1"
ref.set(2) // Logs: "Value changed: 2"

unsubscribe() // Stop listening
```

### Mapping (ReadonlyRef)

```typescript
const numberRef = AtomRef.make(5)

// Create a derived readonly ref
const doubledRef = numberRef.map(n => n * 2)

console.log(doubledRef.value) // 10

numberRef.set(10)
console.log(doubledRef.value) // 20
```

## Collections

For managing arrays of items, `effect-atom` provides a `Collection` type:

```typescript
export interface Collection<A> extends ReadonlyRef<ReadonlyArray<AtomRef<A>>> {
  readonly push: (item: A) => Collection<A>
  readonly insertAt: (index: number, item: A) => Collection<A>
  readonly remove: (ref: AtomRef<A>) => Collection<A>
  readonly toArray: () => Array<A>
}

// Create a collection
const todos = AtomRef.collection([
  { id: 1, text: "Learn effect-atom" },
  { id: 2, text: "Build app" }
])

// Add items
todos.push({ id: 3, text: "Deploy" })

// Each item is an AtomRef for fine-grained updates
todos.value[0].prop("text").set("Master effect-atom")
```

## Implementation Details

The `AtomRef` implementation uses a listener pattern for efficient change propagation:

```typescript
class ReadonlyRefImpl<A> implements ReadonlyRef<A> {
  listeners: Array<(a: A) => void> = []
  listenerCount = 0

  notify(a: A) {
    for (let i = 0; i < this.listenerCount; i++) {
      this.listeners[i](a)
    }
  }

  subscribe(f: (a: A) => void): () => void {
    this.listeners.push(f)
    this.listenerCount++

    return () => {
      const index = this.listeners.indexOf(f)
      if (index !== -1) {
        this.listeners[index] = this.listeners[this.listenerCount - 1]
        this.listeners.pop()
        this.listenerCount--
      }
    }
  }
}
```

The `set` method includes equality checking to prevent unnecessary notifications:

```typescript
set(value: A) {
  if (Equal.equals(value, this.value)) {
    return this
  }
  this.value = value
  this.notify(value)
  return this
}
```

## When to Use Each Primitive

| Use Case | Primitive |
|----------|----------|
| Derived/computed values | `Atom` |
| Async operations with Effect | `Atom` with `Result` |
| Simple mutable state | `AtomRef` |
| Fine-grained object updates | `AtomRef.prop()` |
| List management | `AtomRef.collection()` |
| Cross-cutting reactive computations | `Atom` |

## Related Concepts

- **Registry**: Manages atom instances and their lifecycle
- **Result**: Represents async computation states (waiting, success, failure)
- **Hydration**: Server-side rendering support for atoms
