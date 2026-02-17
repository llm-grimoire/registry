---
title: Effectful Atoms and Result Type
slug: effectful-atoms-and-result-type
description: Working with atoms that perform side effects and using the Result type for handling async state and errors
order: 3
category: patterns
tags: [Result, async, effects, error-handling, state]
relatedFiles: [packages/atom/src/Result.ts, packages/atom/src/Atom.ts]
---

## Overview

When building reactive applications, you often need to handle asynchronous operations like API calls, database queries, or other side effects. The `Result` type in @effect/atom provides a principled way to represent the state of these operations, tracking whether data is loading, has succeeded, or has failed.

This pattern is essential for building robust UIs that need to show loading states, handle errors gracefully, and display data once it arrives.

## The Result Type

The `Result` type is a discriminated union that represents three possible states of an asynchronous operation:

```typescript
import * as Result from "@effect/atom/Result"

// The Result type definition
type Result<A, E = never> = Initial<A, E> | Success<A, E> | Failure<A, E>
```

### Result States

**Initial** - The operation hasn't completed yet (or hasn't started):
```typescript
interface Initial<A, E = never> {
  readonly _tag: "Initial"
  readonly waiting: boolean  // true if currently fetching
}
```

**Success** - The operation completed successfully:
```typescript
interface Success<A, E = never> {
  readonly _tag: "Success"
  readonly value: A
  readonly timestamp: number
  readonly waiting: boolean  // true if refetching
}
```

**Failure** - The operation failed:
```typescript
interface Failure<A, E = never> {
  readonly _tag: "Failure"
  readonly cause: Cause.Cause<E>
  readonly waiting: boolean  // true if retrying
}
```

### The `waiting` Flag

Every Result state includes a `waiting` boolean. This is crucial for representing "stale-while-revalidate" patterns where you have previous data but are fetching fresh data:

```typescript
import * as Result from "@effect/atom/Result"

// Check if any result is currently loading
const isLoading = Result.isWaiting(result) // boolean

// Create an initial waiting state
const loading = Result.initial<User, Error>(true)

// Create a success that's refetching
const staleData = Result.success<User, Error>(user, { waiting: true })
```

## Creating Results

### Basic Constructors

```typescript
import * as Result from "@effect/atom/Result"

// Initial state (not waiting)
const notStarted = Result.initial<string, Error>()

// Initial state (waiting/loading)
const loading = Result.initial<string, Error>(true)

// Success state
const succeeded = Result.success<string, Error>("Hello World")

// Success state with custom timestamp
const withTimestamp = Result.success<string, Error>("Hello", {
  timestamp: Date.now(),
  waiting: false
})
```

### Creating from Effect Exit

When working with Effect, you can convert an `Exit` directly to a Result:

```typescript
import * as Result from "@effect/atom/Result"
import * as Exit from "effect/Exit"

const exit: Exit.Exit<string, Error> = Exit.succeed("data")
const result = Result.fromExit(exit) // Success<string, Error>

// With previous value for failure cases
const resultWithPrevious = Result.fromExitWithPrevious(
  exit,
  Option.some(previousResult)
)
```

### Creating Waiting States from Previous Results

```typescript
import * as Result from "@effect/atom/Result"
import * as Option from "effect/Option"

// If no previous result, returns Initial(waiting: true)
// If previous result exists, returns that result with waiting: true
const waiting = Result.waitingFrom(Option.some(previousResult))
```

## Type Guards and Refinements

```typescript
import * as Result from "@effect/atom/Result"

const result: Result.Result<User, ApiError> = // ...

// Check the result type
if (Result.isResult(unknownValue)) {
  // unknownValue is Result<unknown, unknown>
}

if (Result.isInitial(result)) {
  // result is Initial<User, ApiError>
}

if (Result.isSuccess(result)) {
  // result is Success<User, ApiError>
  console.log(result.value) // User
}

if (Result.isNotInitial(result)) {
  // result is Success<User, ApiError> | Failure<User, ApiError>
}

if (Result.isWaiting(result)) {
  // result.waiting is true (loading/refetching)
}
```

## Working with Results in Atom Context

The Atom `Context` provides special methods for working with Result atoms:

### Getting Results as Effects

```typescript
import * as Atom from "@effect/atom/Atom"
import * as Result from "@effect/atom/Result"
import * as Effect from "effect/Effect"

const userAtom = Atom.make<Result.Result<User, ApiError>>(
  Result.initial()
)

// In an atom read function:
const derivedAtom = Atom.make((get) => {
  // Convert Result to Effect - suspends on Initial/Failure
  // Continues when Success
  return get.result(userAtom)
})

// With suspend on waiting option
const withWaitOption = Atom.make((get) => {
  return get.result(userAtom, { suspendOnWaiting: true })
})
```

### Setting Results

```typescript
const writableResultAtom = Atom.writable<Result.Result<User, ApiError>, User>(
  Result.initial(),
  (ctx, user) => {
    ctx.setSelf(Result.success(user))
  }
)

// Set and wait for success
const setAndWait = Atom.make((get) => {
  // Returns Effect<User, ApiError> that resolves when success
  return get.setResult(writableResultAtom, newUser)
})
```

### Streaming Results

```typescript
const streamAtom = Atom.make((get) => {
  // Stream that emits on success, fails on failure
  // Waits/suspends on Initial
  return get.streamResult(userAtom, {
    withoutInitialValue: false,
    bufferSize: 16
  })
})
```

## Equality and Hashing

Results implement Effect's `Equal` and `Hash` interfaces, making them safe to use in Sets, Maps, and with equality checks:

```typescript
import * as Equal from "effect/Equal"
import * as Result from "@effect/atom/Result"

const result1 = Result.success("hello")
const result2 = Result.success("hello")

Equal.equals(result1, result2) // true

// Results with different waiting states are not equal
const waiting = Result.success("hello", { waiting: true })
Equal.equals(result1, waiting) // false
```

## Type Extraction

Extract success and failure types from atoms:

```typescript
import * as Atom from "@effect/atom/Atom"
import * as Result from "@effect/atom/Result"

const myAtom: Atom.Atom<Result.Result<User, ApiError>> = // ...

type SuccessType = Atom.Success<typeof myAtom> // User
type FailureType = Atom.Failure<typeof myAtom> // ApiError
```

## Best Practices

1. **Use `waiting` for optimistic updates**: When refetching data, set `waiting: true` on your existing Success to show loading indicators while keeping stale data visible.

2. **Leverage `fromExitWithPrevious`**: When an operation fails, preserve the previous successful value so users can still see data.

3. **Use type guards in UI code**: Pattern match on Result states to render appropriate UI for loading, error, and success states.

4. **Combine with streams**: Use `streamResult` to reactively respond to successful values in Effect pipelines.
