---
title: Derived and Computed State
slug: derived-and-computed-state
description: Creating derived atoms that automatically compute values based on other atoms for complex state dependencies
order: 3
category: patterns
tags: [atoms, derived-state, computed, dependencies, reactivity]
relatedFiles: [packages/atom/src/Atom.ts, packages/atom/src/Registry.js]
---

## Overview

Derived and computed state is one of the most powerful features in `effect-atom`. By creating atoms that read from other atoms, you can build complex dependency graphs where computed values automatically update when their dependencies change. This reactive pattern eliminates manual synchronization and ensures your state remains consistent.

## Key Concepts

### The Read Function and Context

Every atom receives a `read` function that takes a `Context` parameter. This context provides the `get` method to read other atoms, establishing automatic dependency tracking:

```typescript
import { Atom } from "effect-atom"

// Base atoms
const firstName = Atom.make("John")
const lastName = Atom.make("Doe")

// Derived atom - automatically recomputes when firstName or lastName changes
const fullName = Atom.make((get) => {
  const first = get(firstName)
  const last = get(lastName)
  return `${first} ${last}`
})
```

### The Context Interface

The `Context` interface provides several methods for reading and interacting with atoms:

```typescript
export interface Context {
  // Read an atom's value (establishes dependency)
  <A>(atom: Atom<A>): A
  get<A>(this: Context, atom: Atom<A>): A
  
  // Read once without establishing dependency
  once<A>(this: Context, atom: Atom<A>): A
  
  // Read Result atoms with Effect handling
  result<A, E>(this: Context, atom: Atom<Result.Result<A, E>>, options?: {
    readonly suspendOnWaiting?: boolean | undefined
  }): Effect.Effect<A, E>
  
  // Refresh dependencies
  refresh<A>(this: Context, atom: Atom<A>): void
  refreshSelf(this: Context): void
  
  // Access the registry
  readonly registry: Registry.Registry
}
```

## Creating Derived Atoms

### Basic Computed Values

```typescript
import { Atom } from "effect-atom"

// Source atoms
const items = Atom.make<Array<{ price: number; quantity: number }>>([])
const taxRate = Atom.make(0.08)

// Derived: subtotal
const subtotal = Atom.make((get) => {
  const cartItems = get(items)
  return cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0)
})

// Derived: tax amount (depends on subtotal and taxRate)
const taxAmount = Atom.make((get) => {
  return get(subtotal) * get(taxRate)
})

// Derived: total (depends on subtotal and taxAmount)
const total = Atom.make((get) => {
  return get(subtotal) + get(taxAmount)
})
```

### Reading Without Dependency Tracking

Use `once` when you need to read a value without establishing a reactive dependency:

```typescript
const configAtom = Atom.make({ defaultCurrency: "USD" })
const priceAtom = Atom.make(100)

// This atom only recomputes when priceAtom changes, not when configAtom changes
const formattedPrice = Atom.make((get) => {
  const price = get(priceAtom) // Reactive dependency
  const config = get.once(configAtom) // Read once, no dependency
  return `${price} ${config.defaultCurrency}`
})
```

### Conditional Dependencies

Dependencies are tracked dynamically based on code execution:

```typescript
const showDetails = Atom.make(false)
const basicInfo = Atom.make({ name: "Product" })
const detailedInfo = Atom.make({ name: "Product", specs: "..." })

// Only depends on detailedInfo when showDetails is true
const displayInfo = Atom.make((get) => {
  if (get(showDetails)) {
    return get(detailedInfo)
  }
  return get(basicInfo)
})
```

## Working with Result Types

Derived atoms can work with async Result types using the `result` method:

```typescript
import { Atom, Result } from "effect-atom"
import * as Effect from "effect/Effect"

const userId = Atom.make(1)

// Async atom that fetches user data
const userData = Atom.effect((get, signal) => 
  Effect.gen(function* () {
    const id = get(userId)
    const response = yield* Effect.tryPromise({
      try: () => fetch(`/api/users/${id}`, { signal }),
      catch: (e) => new Error("Failed to fetch user")
    })
    return yield* Effect.tryPromise({
      try: () => response.json(),
      catch: (e) => new Error("Failed to parse response")
    })
  })
)

// Derived atom that computes from async result
const userDisplayName = Atom.effect((get) =>
  Effect.gen(function* () {
    const user = yield* get.result(userData)
    return `${user.firstName} ${user.lastName}`
  })
)
```

## Streaming Derived Values

The context provides `stream` for observing atom changes reactively:

```typescript
const counter = Atom.make(0)

const loggedCounter = Atom.make((get) => {
  // Subscribe to counter changes within this atom's lifecycle
  get.subscribe(counter, (value) => {
    console.log(`Counter changed to: ${value}`)
  })
  
  return get(counter)
})
```

## Self-Referential Atoms

Atoms can access and modify their own previous value:

```typescript
const accumulator = Atom.make((get) => {
  const previous = get.self<number>() // Option<number>
  return previous._tag === "Some" ? previous.value : 0
})
```

## Best Practices

1. **Keep derivations pure**: Derived atoms should be pure computations without side effects
2. **Use `once` for static config**: Prevent unnecessary recomputation for values that rarely change
3. **Minimize dependency chains**: Deep dependency chains can impact performance
4. **Leverage conditional dependencies**: Only read atoms when their values are actually needed
5. **Use Result types for async**: Properly handle loading and error states in derived async atoms

## Related Patterns

- **Atom Families**: Create parameterized derived atoms for dynamic keys
- **Effect Integration**: Use `Atom.effect` for derived atoms with async computations
- **Registry**: Manage derived atom lifecycle and cleanup
