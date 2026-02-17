---
title: LiveStore Integration
slug: livestore-integration
description: Connecting atoms with LiveStore for persistent, synchronized reactive state management across clients
order: 6
category: guides
tags: [livestore, persistence, sync, reactive, state-management, real-time]
relatedFiles: [packages/atom-livestore/src/index.ts]
---

## Overview

The `@livestore/atom-livestore` package provides integration between Effect's reactive atom system and LiveStore, enabling persistent and synchronized state management across multiple clients. This integration allows you to create reactive atoms that automatically sync with LiveStore's real-time database, making it ideal for collaborative applications, offline-first apps, and scenarios requiring state persistence.

## Key Concepts

### What is LiveStore?

LiveStore is a real-time, reactive database that synchronizes state across clients. When combined with Effect's atom system, you get:

- **Reactive State**: Atoms that automatically update when underlying data changes
- **Persistence**: State that survives page reloads and app restarts
- **Synchronization**: Real-time sync across multiple clients/devices
- **Conflict Resolution**: Built-in handling for concurrent updates

### AtomLivestore Module

The integration is exposed through the `AtomLivestore` namespace:

```typescript
import { AtomLivestore } from "@livestore/atom-livestore"
```

## Installation

```bash
npm install @livestore/atom-livestore @livestore/livestore
```

## Basic Usage

### Creating a LiveStore-Backed Atom

```typescript
import { AtomLivestore } from "@livestore/atom-livestore"
import { Effect } from "effect"

// Create an atom that syncs with LiveStore
const userPreferences = AtomLivestore.make({
  key: "user-preferences",
  defaultValue: {
    theme: "light",
    language: "en",
    notifications: true
  }
})

// Use the atom like any other reactive atom
const program = Effect.gen(function* () {
  const prefs = yield* userPreferences.get
  console.log("Current theme:", prefs.theme)
  
  // Updates automatically sync to LiveStore
  yield* userPreferences.set({
    ...prefs,
    theme: "dark"
  })
})
```

### Subscribing to Changes

```typescript
import { AtomLivestore } from "@livestore/atom-livestore"
import { Effect, Stream } from "effect"

const counter = AtomLivestore.make({
  key: "shared-counter",
  defaultValue: 0
})

// Subscribe to real-time updates from all clients
const subscription = counter.changes.pipe(
  Stream.tap((value) => 
    Effect.log(`Counter changed to: ${value}`)
  ),
  Stream.runDrain
)
```

## Advanced Patterns

### Scoped LiveStore Atoms

```typescript
import { AtomLivestore } from "@livestore/atom-livestore"
import { Effect } from "effect"

// Create atoms scoped to specific contexts
const createRoomAtom = (roomId: string) =>
  AtomLivestore.make({
    key: `room:${roomId}:messages`,
    defaultValue: [] as Message[]
  })

const program = Effect.gen(function* () {
  const room123Messages = createRoomAtom("123")
  
  // This atom only syncs messages for room 123
  const messages = yield* room123Messages.get
})
```

### Combining with Derived Atoms

```typescript
import { AtomLivestore } from "@livestore/atom-livestore"
import { Atom } from "@effect/core"
import { Effect } from "effect"

// Base persisted atom
const tasks = AtomLivestore.make({
  key: "tasks",
  defaultValue: [] as Task[]
})

// Derived atoms computed from persisted state
const completedTasks = Atom.derived(
  tasks,
  (allTasks) => allTasks.filter(t => t.completed)
)

const pendingCount = Atom.derived(
  tasks,
  (allTasks) => allTasks.filter(t => !t.completed).length
)
```

## Architecture Considerations

### When to Use LiveStore Integration

- **Collaborative editing**: Multiple users editing shared documents
- **Real-time dashboards**: Live data that updates across all viewers
- **Offline-first apps**: State that persists and syncs when online
- **Cross-device sync**: User preferences and data across devices

### Performance Tips

1. **Granular keys**: Use specific keys rather than storing large objects
2. **Batch updates**: Group related updates to reduce sync overhead
3. **Selective persistence**: Only persist state that truly needs sync

## Related Resources

- See the `@effect/core` atom documentation for base atom concepts
- Explore LiveStore documentation for database configuration
- Check pattern guides for reactive state management best practices
