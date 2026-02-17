---
title: Installation and Setup
slug: installation-and-setup
description: Guide to installing the atom library and framework-specific bindings in your project using pnpm workspaces
order: 1
category: setup
tags: [installation, setup, pnpm, react, effect, getting-started]
relatedFiles: [package.json, packages/atom/package.json, packages/atom-react/package.json]
---

## Overview

This guide covers how to install and set up the Effect Atom library in your project. Effect Atom is a reactive toolkit built on top of Effect, providing powerful state management primitives. The library is organized as a monorepo with the core `@effect-atom/atom` package and framework-specific bindings like `@effect-atom/atom-react`.

## Prerequisites

Before installing Effect Atom, ensure you have:

- Node.js (compatible with your Effect version)
- pnpm (the project uses pnpm 10.18.1+)
- Effect library (`^3.19.15` or compatible)

## Installing the Core Package

The core atom library provides the fundamental reactive primitives. Install it along with its peer dependencies:

```bash
pnpm add @effect-atom/atom effect @effect/platform @effect/experimental @effect/rpc
```

### Peer Dependencies

The core package requires the following peer dependencies:

```json
{
  "peerDependencies": {
    "@effect/experimental": "^0.58.0",
    "@effect/platform": "^0.94.2",
    "@effect/rpc": "^0.73.0",
    "effect": "^3.19.15"
  }
}
```

Make sure to install compatible versions of these packages in your project.

## Installing React Bindings

If you're using React, install the React-specific bindings:

```bash
pnpm add @effect-atom/atom-react react scheduler
```

### React Peer Dependencies

The React package requires:

```json
{
  "peerDependencies": {
    "effect": "^3.19",
    "react": ">=18 <20",
    "scheduler": "*"
  }
}
```

Note that `@effect-atom/atom-react` includes `@effect-atom/atom` as a direct dependency, so you don't need to install the core package separately when using React bindings.

## Monorepo Setup

If you're contributing to Effect Atom or setting up a similar monorepo structure, the project uses pnpm workspaces:

```json
{
  "private": true,
  "packageManager": "pnpm@10.18.1",
  "workspaces": [
    "packages/*"
  ]
}
```

### Building from Source

To build the project from source:

```bash
# Clone the repository
git clone https://github.com/tim-smart/effect-atom.git
cd effect-atom

# Install dependencies
pnpm install

# Build all packages
pnpm build
```

### Development Scripts

The monorepo provides several useful scripts:

```bash
# Run tests
pnpm test

# Run tests with coverage
pnpm coverage

# Type checking
pnpm check

# Linting
pnpm lint

# Fix linting issues
pnpm lint-fix

# Check for circular dependencies
pnpm circular

# Clean build artifacts
pnpm clean
```

## TypeScript Configuration

Both packages are built with TypeScript and provide full type definitions. The build process generates:

- ESM modules (primary)
- CommonJS modules (for compatibility)
- Source maps for debugging

Ensure your `tsconfig.json` has `moduleResolution` set to `bundler` or `node16`/`nodenext` for proper module resolution:

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "target": "ES2022",
    "module": "ESNext",
    "strict": true
  }
}
```

## Verifying Installation

After installation, verify everything is working by importing the core module:

```typescript
import { Atom, Computed, AtomRegistry } from "@effect-atom/atom"
import { Effect } from "effect"

// If using React
import { useAtom, useComputed, AtomProvider } from "@effect-atom/atom-react"
```

If the imports resolve without errors, you're ready to start using Effect Atom!

## Next Steps

Once installed, explore:

- **Atom basics**: Learn about creating and managing reactive state with `Atom`
- **Computed values**: Derive state with `Computed`
- **React integration**: Use atoms in React components with hooks
- **AtomRegistry**: Manage atom lifecycles and subscriptions
