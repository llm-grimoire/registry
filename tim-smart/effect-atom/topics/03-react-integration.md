---
title: React Integration Guide
slug: react-integration
description: Complete guide to using atoms in React applications with hooks like useAtomValue, useAtomSet, and RegistryContext
order: 3
category: guides
tags: [react, hooks, context, integration, frontend]
relatedFiles: [packages/atom-react/src/index.ts, packages/atom-react/src/hooks.ts, packages/atom-react/src/RegistryContext.ts]
---

## Overview

The `@effect-atom/atom-react` package provides seamless integration between Effect Atom and React applications. It offers a set of React hooks and context providers that enable components to read, write, and subscribe to atom state changes with proper lifecycle management.

The integration is built on React's `useSyncExternalStore` for optimal performance and compatibility with React's concurrent features, including server-side rendering support.

## Key Concepts

### Registry Context

The `RegistryContext` is the foundation of React integration. It provides a Registry instance to all components in your application tree, enabling them to access and modify atom state.

### Automatic Subscription Management

When you use hooks like `useAtomValue`, the library automatically:
- Subscribes to atom changes when the component mounts
- Re-renders the component when the atom value changes
- Unsubscribes when the component unmounts

### Mount Lifecycle

Atoms can define mount/unmount behavior (like starting timers or subscriptions). The `useAtomMount` and `useAtomSet` hooks ensure atoms are properly mounted during the component lifecycle.

## Setting Up the Provider

Wrap your application with `RegistryProvider` to establish the atom registry context:

```tsx
import { RegistryProvider } from "@effect-atom/atom-react"

function App() {
  return (
    <RegistryProvider
      initialValues={[
        [userAtom, { name: "Guest" }],
        [themeAtom, "dark"]
      ]}
      defaultIdleTTL={400}
    >
      <YourApplication />
    </RegistryProvider>
  )
}
```

The `RegistryProvider` accepts several configuration options:

- `initialValues` - Pre-populate atoms with initial values
- `scheduleTask` - Custom task scheduler (defaults to React's scheduler)
- `timeoutResolution` - Resolution for internal timeouts
- `defaultIdleTTL` - Time-to-live for idle atom nodes

## Reading Atom Values

### Basic Usage with useAtomValue

The `useAtomValue` hook subscribes to an atom and returns its current value:

```tsx
import { useAtomValue, Atom } from "@effect-atom/atom-react"

const countAtom = Atom.state(0)

function Counter() {
  const count = useAtomValue(countAtom)
  return <span>Count: {count}</span>
}
```

### Derived Values with Selector

You can pass a selector function to transform the atom value:

```tsx
import { useAtomValue } from "@effect-atom/atom-react"

function DoubledCounter() {
  const doubled = useAtomValue(countAtom, (count) => count * 2)
  return <span>Doubled: {doubled}</span>
}
```

The selector is memoized, so the derived atom is only recreated when the atom or selector function changes.

## Writing Atom Values

### Basic Updates with useAtomSet

The `useAtomSet` hook returns a setter function for writable atoms:

```tsx
import { useAtomSet, Atom } from "@effect-atom/atom-react"

const countAtom = Atom.state(0)

function IncrementButton() {
  const setCount = useAtomSet(countAtom)
  
  return (
    <button onClick={() => setCount((prev) => prev + 1)}>
      Increment
    </button>
  )
}
```

The setter supports both direct values and updater functions:

```tsx
setCount(5)                    // Set to 5
setCount((prev) => prev + 1)   // Increment by 1
```

### Promise Mode for Async Atoms

For atoms that return `Result` types (async operations), you can use promise mode:

```tsx
import { useAtomSet, Atom, Result } from "@effect-atom/atom-react"

const fetchUserAtom = Atom.writable<Result.Result<User, Error>, string>({
  read: (get) => get(userResultAtom),
  write: (get, set, userId) => {
    // Trigger fetch
  }
})

function UserLoader() {
  const fetchUser = useAtomSet(fetchUserAtom, { mode: "promise" })
  
  const handleFetch = async () => {
    try {
      const user = await fetchUser("user-123")
      console.log("Loaded:", user)
    } catch (error) {
      console.error("Failed:", error)
    }
  }
  
  return <button onClick={handleFetch}>Load User</button>
}
```

Available modes:
- `"value"` (default) - Synchronous setter
- `"promise"` - Returns a Promise that resolves with the success value or rejects on failure
- `"promiseExit"` - Returns a Promise with the full Exit containing success or failure

## Combined Read/Write with useAtom

The `useAtom` hook combines reading and writing into a single call:

```tsx
import { useAtom, Atom } from "@effect-atom/atom-react"

const nameAtom = Atom.state("")

function NameInput() {
  const [name, setName] = useAtom(nameAtom)
  
  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  )
}
```

## Mounting Atoms

### Manual Mount with useAtomMount

Some atoms have side effects that run on mount (like subscriptions or timers). Use `useAtomMount` to ensure proper lifecycle:

```tsx
import { useAtomMount, useAtomValue } from "@effect-atom/atom-react"

const timerAtom = Atom.state(0, {
  onMount: (set) => {
    const id = setInterval(() => set((n) => n + 1), 1000)
    return () => clearInterval(id)
  }
})

function Timer() {
  useAtomMount(timerAtom)  // Ensures timer starts
  const seconds = useAtomValue(timerAtom)
  
  return <span>Elapsed: {seconds}s</span>
}
```

Note: `useAtomSet` automatically mounts the atom, so you don't need both hooks when writing.

## Setting Initial Values

### useAtomInitialValues

For hydration scenarios or setting initial values imperatively:

```tsx
import { useAtomInitialValues } from "@effect-atom/atom-react"

function HydratedApp({ serverData }) {
  useAtomInitialValues([
    [userAtom, serverData.user],
    [settingsAtom, serverData.settings]
  ])
  
  return <App />
}
```

This hook ensures values are only set once per atom, preventing overwrites on re-renders.

## Server-Side Rendering

The hooks support SSR through `getServerSnapshot`. Atoms can provide server-specific values using `Atom.getServerValue`:

```tsx
const store: AtomStore<A> = {
  subscribe(f) {
    return registry.subscribe(atom, f)
  },
  snapshot() {
    return registry.get(atom)
  },
  getServerSnapshot() {
    return Atom.getServerValue(atom, registry)
  }
}
```

This ensures consistent rendering between server and client during hydration.

## Re-exports

The `@effect-atom/atom-react` package re-exports core modules for convenience:

```tsx
import { 
  Atom,      // Core atom creation and utilities
  Registry,  // Registry management
  Result,    // Async result types
  AtomRef,   // Reference atoms
  AtomHttpApi,  // HTTP API integration
  AtomRpc,   // RPC integration
  Hydration, // Server hydration utilities
  ScopedAtom // Scoped atom patterns
} from "@effect-atom/atom-react"
```

## Best Practices

1. **Place RegistryProvider high in the tree** - Typically at the app root to share state across all components

2. **Use selectors for derived state** - Pass a selector to `useAtomValue` instead of creating intermediate atoms for simple transformations

3. **Prefer useAtomSet for write-only components** - It's more efficient than `useAtom` when you don't need to read the value

4. **Handle async atoms with promise mode** - Use `{ mode: "promise" }` or `{ mode: "promiseExit" }` for proper async error handling

5. **Initialize atoms declaratively** - Use `initialValues` prop on `RegistryProvider` when possible instead of imperative `useAtomInitialValues`
