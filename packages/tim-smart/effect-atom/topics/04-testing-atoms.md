---
title: Testing Atoms
slug: testing-atoms
description: Best practices and patterns for unit testing atoms and components that depend on atomic state
order: 4
category: patterns
tags: [testing, atoms, registry, unit-tests, mocking, isolation]
relatedFiles: [packages/atom/src/Registry.ts, packages/atom/src/Atom.ts, packages/atom/src/Result.ts]
---

## Overview

Testing atoms effectively requires understanding how to create isolated registry instances, provide initial values, and properly clean up after tests. The `@effect/atom` library provides several mechanisms specifically designed to make testing straightforward and reliable.

This guide covers strategies for unit testing atoms, testing components that depend on atomic state, and handling asynchronous atom behaviors in test environments.

## Key Concepts

### Test Isolation with Fresh Registries

The most important principle when testing atoms is **isolation**. Each test should have its own `Registry` instance to prevent state leakage between tests. The `Registry.make()` constructor creates a fresh registry that can be configured specifically for testing needs.

### Initial Values for Deterministic Tests

Atoms can be pre-populated with specific values using the `initialValues` option, allowing you to set up precise test scenarios without triggering atom computations.

### Controlled Task Scheduling

For testing asynchronous behaviors, you can provide a custom `scheduleTask` function to control when deferred updates occur.

## Creating Test Registries

### Basic Test Registry

Create a fresh registry for each test to ensure complete isolation:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

describe("Counter Atom", () => {
  let registry: Registry.Registry

  beforeEach(() => {
    registry = Registry.make()
  })

  afterEach(() => {
    registry.dispose()
  })

  it("should initialize with default value", () => {
    const counterAtom = Atom.make(0)
    expect(registry.get(counterAtom)).toBe(0)
  })
})
```

### Pre-populating State with Initial Values

Use the `initialValues` option to set up specific test scenarios:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

const userAtom = Atom.make<User | null>(null)
const settingsAtom = Atom.make({ theme: "light", notifications: true })

describe("User Settings", () => {
  it("should handle logged-in user scenario", () => {
    const testUser: User = { id: "123", name: "Test User" }
    
    const registry = Registry.make({
      initialValues: [
        [userAtom, testUser],
        [settingsAtom, { theme: "dark", notifications: false }]
      ]
    })

    expect(registry.get(userAtom)).toEqual(testUser)
    expect(registry.get(settingsAtom).theme).toBe("dark")
    
    registry.dispose()
  })
})
```

## Testing Derived Atoms

Derived atoms that compute values from other atoms can be tested by setting up their dependencies:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

const priceAtom = Atom.make(100)
const quantityAtom = Atom.make(2)
const totalAtom = Atom.derived((get) => get(priceAtom) * get(quantityAtom))

describe("Total Calculation", () => {
  it("should compute total from price and quantity", () => {
    const registry = Registry.make({
      initialValues: [
        [priceAtom, 50],
        [quantityAtom, 3]
      ]
    })

    expect(registry.get(totalAtom)).toBe(150)
    
    registry.dispose()
  })

  it("should update when dependencies change", () => {
    const registry = Registry.make()

    expect(registry.get(totalAtom)).toBe(200) // 100 * 2
    
    registry.set(quantityAtom, 5)
    expect(registry.get(totalAtom)).toBe(500) // 100 * 5
    
    registry.dispose()
  })
})
```

## Testing Subscriptions

Test that atoms properly notify subscribers when values change:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

const countAtom = Atom.make(0)

describe("Atom Subscriptions", () => {
  it("should notify subscribers on change", () => {
    const registry = Registry.make()
    const values: number[] = []

    const unsubscribe = registry.subscribe(
      countAtom,
      (value) => values.push(value),
      { immediate: true }
    )

    registry.set(countAtom, 1)
    registry.set(countAtom, 2)
    registry.set(countAtom, 3)

    expect(values).toEqual([0, 1, 2, 3])
    
    unsubscribe()
    registry.dispose()
  })

  it("should stop notifying after unsubscribe", () => {
    const registry = Registry.make()
    const values: number[] = []

    const unsubscribe = registry.subscribe(
      countAtom,
      (value) => values.push(value)
    )

    registry.set(countAtom, 1)
    unsubscribe()
    registry.set(countAtom, 2)

    expect(values).toEqual([1])
    
    registry.dispose()
  })
})
```

## Testing with Effect Layer

For Effect-based applications, use `layerOptions` to create a properly scoped registry:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"
import * as Effect from "effect/Effect"
import * as Layer from "effect/Layer"

const testAtom = Atom.make("initial")

describe("Effect Integration", () => {
  it("should work with Effect runtime", async () => {
    const testLayer = Registry.layerOptions({
      initialValues: [[testAtom, "test value"]]
    })

    const program = Effect.gen(function* () {
      const registry = yield* Registry.AtomRegistry
      return registry.get(testAtom)
    })

    const result = await Effect.runPromise(
      program.pipe(Effect.provide(testLayer))
    )

    expect(result).toBe("test value")
  })
})
```

## Testing Async Atoms with Controlled Scheduling

Control when scheduled tasks execute for deterministic async testing:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

describe("Async Atom Behavior", () => {
  it("should allow controlled task scheduling", () => {
    const pendingTasks: Array<() => void> = []
    
    const registry = Registry.make({
      scheduleTask: (f) => pendingTasks.push(f)
    })

    const asyncAtom = Atom.make(0)
    
    // Perform operations that schedule tasks
    registry.set(asyncAtom, 1)
    
    // Manually flush pending tasks
    while (pendingTasks.length > 0) {
      const task = pendingTasks.shift()!
      task()
    }
    
    expect(registry.get(asyncAtom)).toBe(1)
    
    registry.dispose()
  })
})
```

## Testing Registry Reset

The `reset()` method allows testing state restoration:

```typescript
import * as Registry from "@effect/atom/Registry"
import * as Atom from "@effect/atom/Atom"

const stateAtom = Atom.make({ count: 0, name: "" })

describe("Registry Reset", () => {
  it("should reset all atoms to initial values", () => {
    const registry = Registry.make()

    registry.set(stateAtom, { count: 10, name: "modified" })
    expect(registry.get(stateAtom).count).toBe(10)

    registry.reset()
    expect(registry.get(stateAtom)).toEqual({ count: 0, name: "" })
    
    registry.dispose()
  })
})
```

## Best Practices

1. **Always dispose registries** - Call `registry.dispose()` in `afterEach` to prevent memory leaks
2. **Use `initialValues` for complex setups** - Avoid multiple `set` calls by pre-populating state
3. **Test subscriptions with `immediate: true`** - Capture the initial value to verify starting state
4. **Isolate tests completely** - Never share registry instances between test cases
5. **Use custom schedulers for async control** - Collect scheduled tasks and flush them manually for deterministic testing
6. **Test derived atoms through their dependencies** - Set up source atoms and verify computed results
