---
title: Vue Integration Guide
slug: vue-integration
description: Integrating atoms into Vue applications with reactive bindings and composition API support
order: 3
category: guides
tags: [vue, integration, composables, reactive, composition-api]
relatedFiles: [packages/atom-vue/src/index.ts]
---

## Overview

The `@effect-atom/atom-vue` package provides seamless integration between Effect Atom and Vue 3 applications. It leverages Vue's Composition API to create reactive bindings that automatically update your components when atom state changes.

This integration enables you to:
- Use atoms as reactive state in Vue components
- Automatically subscribe and unsubscribe from atom updates
- Handle async operations with promise-based writes
- Share state across components via dependency injection

## Key Concepts

### Registry and Dependency Injection

The Vue integration uses Vue's dependency injection system to provide the atom registry to components. A default registry is available globally, but you can provide custom registries for testing or isolation.

```typescript
import { registryKey, defaultRegistry, injectRegistry } from "@effect-atom/atom-vue"
import { provide } from "vue"

// In your root component or plugin
provide(registryKey, defaultRegistry)

// In any child component
const registry = injectRegistry()
```

### Composables

The package exports three main composables that follow Vue's naming conventions:

1. **`useAtom`** - Read and write an atom's value
2. **`useAtomValue`** - Read-only access to an atom's value
3. **`useAtomSet`** - Write-only access to an atom

## Usage Examples

### Basic Atom Usage

Use `useAtomValue` for read-only access to atom state:

```typescript
import { useAtomValue } from "@effect-atom/atom-vue"
import { countAtom } from "./atoms"

// In your setup function
const count = useAtomValue(() => countAtom)
// count is Readonly<Ref<number>> - fully reactive!
```

### Reading and Writing Atoms

Use `useAtom` to get both read and write access:

```typescript
import { useAtom } from "@effect-atom/atom-vue"
import { countAtom } from "./atoms"

// Returns a tuple of [value, setter]
const [count, setCount] = useAtom(() => countAtom)

// Direct value
setCount(10)

// Updater function
setCount((current) => current + 1)
```

### Async Operations with Promise Mode

For atoms that return `Result` types, you can use promise modes to await the completion:

```typescript
import { useAtom } from "@effect-atom/atom-vue"
import { asyncDataAtom } from "./atoms"

// Promise mode - throws on error
const [data, fetchData] = useAtom(() => asyncDataAtom, { mode: "promise" })

async function handleSubmit(params: SubmitParams) {
  try {
    const result = await fetchData(params)
    console.log("Success:", result)
  } catch (error) {
    console.error("Failed:", error)
  }
}
```

### Promise Exit Mode

For more control over error handling, use `promiseExit` mode which returns an Effect `Exit`:

```typescript
import { useAtom } from "@effect-atom/atom-vue"
import * as Exit from "effect/Exit"
import { asyncDataAtom } from "./atoms"

const [data, fetchData] = useAtom(() => asyncDataAtom, { mode: "promiseExit" })

async function handleSubmit(params: SubmitParams) {
  const exit = await fetchData(params)
  
  if (Exit.isSuccess(exit)) {
    console.log("Success:", exit.value)
  } else {
    console.log("Failure:", exit.cause)
  }
}
```

### Write-Only Access

Use `useAtomSet` when you only need to write to an atom:

```typescript
import { useAtomSet } from "@effect-atom/atom-vue"
import { settingsAtom } from "./atoms"

const updateSettings = useAtomSet(() => settingsAtom)

// Call anywhere in your component
updateSettings({ theme: "dark" })
```

### Abort Signal Support

Promise modes support cancellation via AbortSignal:

```typescript
const [data, fetchData] = useAtom(() => asyncDataAtom, { mode: "promise" })

const controller = new AbortController()

fetchData(params, { signal: controller.signal })

// Later, to cancel:
controller.abort()
```

## Module Re-exports

The Vue package re-exports all core modules for convenience:

```typescript
import {
  Atom,
  AtomRef,
  Registry,
  Result,
  AtomHttpApi,
  AtomRpc
} from "@effect-atom/atom-vue"
```

This allows you to import everything from a single package in Vue applications.

## Reactivity Implementation

Under the hood, the integration uses:
- `shallowRef` for storing atom values (efficient for complex objects)
- `computed` for memoizing the atom reference
- `watchEffect` for automatic cleanup when components unmount

```typescript
// Internal implementation pattern
const useAtomValueRef = <A extends Atom.Atom<any>>(atom: () => A) => {
  const registry = injectRegistry()
  const atomRef = computed(atom)
  const value = shallowRef(undefined as any as A)
  
  watchEffect((onCleanup) => {
    onCleanup(registry.subscribe(atomRef.value, (nextValue) => {
      value.value = nextValue
    }, { immediate: true }))
  })
  
  return [value, atomRef, registry] as const
}
```

This ensures subscriptions are properly cleaned up when components are destroyed or when the atom reference changes.
