---
title: Scoped Atoms in React
slug: scoped-atoms-react
description: Managing component-scoped state with ScopedAtom for isolated state lifecycles tied to React component trees
order: 4
category: patterns
tags: [react, scoped-atom, component-state, provider-pattern, isolation]
relatedFiles: [packages/atom-react/src/ScopedAtom.ts, packages/atom-react/src/index.ts]
---

## Overview

ScopedAtom provides a pattern for creating atoms whose lifecycle is tied to a specific React component tree. Unlike global atoms that persist throughout the application, scoped atoms are created fresh for each Provider instance and are isolated from other parts of the application.

This pattern is essential when you need:
- Multiple independent instances of the same state structure
- State that should be garbage collected when components unmount
- Isolated state for modal dialogs, forms, or feature modules
- Testing components with isolated state

## Key Concepts

### ScopedAtom Interface

A ScopedAtom wraps an atom factory function and provides React integration:

```typescript
import type * as Atom from "@effect-atom/atom/Atom"
import * as React from "react"

export interface ScopedAtom<A extends Atom.Atom<any>, Input = never> {
  readonly [TypeId]: TypeId
  use(): A
  Provider: Input extends never 
    ? React.FC<{ readonly children?: React.ReactNode | undefined }>
    : React.FC<{ readonly children?: React.ReactNode | undefined; readonly value: Input }>
  Context: React.Context<A>
}
```

The interface exposes three key members:
- **`use()`** - A hook to access the scoped atom within the provider tree
- **`Provider`** - A React component that creates and provides the atom instance
- **`Context`** - The underlying React context (for advanced use cases)

### Creating a ScopedAtom

Use `ScopedAtom.make()` to create a scoped atom from a factory function:

```typescript
import { ScopedAtom, Atom } from "@effect-atom/atom-react"

// Simple scoped atom without input
const CounterAtom = ScopedAtom.make(() => 
  Atom.make({ count: 0 })
)

// Scoped atom with input parameter
const UserFormAtom = ScopedAtom.make((initialUser: { name: string; email: string }) =>
  Atom.make(initialUser)
)
```

### Lazy Initialization with useRef

The Provider implementation uses `React.useRef` to ensure the atom is only created once per Provider instance:

```typescript
const Provider: React.FC<{ readonly children?: React.ReactNode; readonly value: Input }> = ({
  children,
  value
}) => {
  const atom = React.useRef<A | null>(null)
  if (atom.current === null) {
    atom.current = f(value)
  }
  return React.createElement(Context.Provider, { value: atom.current }, children)
}
```

This pattern ensures:
- The atom is created synchronously on first render
- Re-renders don't recreate the atom
- The atom persists for the lifetime of the Provider component

## Usage Examples

### Basic Counter with Scoped State

```typescript
import { ScopedAtom, Atom, useAtomValue, useSetAtom } from "@effect-atom/atom-react"

// Define the scoped atom
const CounterAtom = ScopedAtom.make(() => 
  Atom.make({ count: 0 })
)

// Counter display component
function CounterDisplay() {
  const counter = CounterAtom.use()
  const { count } = useAtomValue(counter)
  return <span>Count: {count}</span>
}

// Counter controls component  
function CounterControls() {
  const counter = CounterAtom.use()
  const setCounter = useSetAtom(counter)
  
  return (
    <button onClick={() => setCounter(prev => ({ count: prev.count + 1 }))}>
      Increment
    </button>
  )
}

// Multiple independent counter instances
function App() {
  return (
    <div>
      {/* Each Provider creates an isolated counter */}
      <CounterAtom.Provider>
        <h2>Counter A</h2>
        <CounterDisplay />
        <CounterControls />
      </CounterAtom.Provider>
      
      <CounterAtom.Provider>
        <h2>Counter B</h2>
        <CounterDisplay />
        <CounterControls />
      </CounterAtom.Provider>
    </div>
  )
}
```

### Form State with Input Parameters

```typescript
import { ScopedAtom, Atom, useAtomValue, useSetAtom } from "@effect-atom/atom-react"

interface FormData {
  name: string
  email: string
  submitted: boolean
}

// Scoped atom that accepts initial form data
const FormAtom = ScopedAtom.make((initial: Omit<FormData, 'submitted'>) =>
  Atom.make<FormData>({ ...initial, submitted: false })
)

function FormFields() {
  const form = FormAtom.use()
  const data = useAtomValue(form)
  const setForm = useSetAtom(form)
  
  return (
    <form>
      <input
        value={data.name}
        onChange={e => setForm(prev => ({ ...prev, name: e.target.value }))}
      />
      <input
        value={data.email}
        onChange={e => setForm(prev => ({ ...prev, email: e.target.value }))}
      />
    </form>
  )
}

// Each modal gets its own isolated form state
function EditUserModal({ user }: { user: { name: string; email: string } }) {
  return (
    <FormAtom.Provider value={user}>
      <FormFields />
    </FormAtom.Provider>
  )
}
```

### Error Handling

Using a scoped atom outside its Provider throws a descriptive error:

```typescript
const use = (): A => {
  const atom = React.useContext(Context)
  if (atom === undefined) {
    throw new Error("ScopedAtom used outside of its Provider")
  }
  return atom
}
```

This helps catch configuration errors during development:

```typescript
// This will throw an error
function BrokenComponent() {
  const counter = CounterAtom.use() // Error: ScopedAtom used outside of its Provider
  return <span>{counter}</span>
}
```

## When to Use Scoped vs Global Atoms

| Use Case | Recommendation |
|----------|----------------|
| App-wide settings, theme, auth | Global Atom |
| Form state in modals | ScopedAtom |
| List item state (multiple instances) | ScopedAtom |
| Cache/data fetching | Global Atom with Registry |
| Wizard/multi-step flow state | ScopedAtom |
| Shopping cart | Global Atom |

## Related Patterns

- Use ScopedAtom with derived atoms for computed values within the scope
- Combine with Effect for async operations scoped to the component tree
- Access the underlying `Context` for integration with other React patterns
