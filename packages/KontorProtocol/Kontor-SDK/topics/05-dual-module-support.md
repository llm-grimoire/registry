---
title: Dual Module System (CJS/ESM)
slug: dual-module-support
description: Understanding the dual CommonJS and ES Module build output and usage patterns
order: 5
category: architecture
tags: [modules, esm, cjs, build, typescript, exports]
relatedFiles: [tsconfig.json, src/exports/index.ts]
---

## Overview

The Kontor SDK is built with dual module support, providing both CommonJS (CJS) and ES Module (ESM) outputs. This ensures compatibility across different JavaScript environments, from Node.js applications to modern bundlers and browsers.

The dual module system allows developers to:
- Use modern `import` statements in ESM environments
- Use `require()` in CommonJS environments
- Benefit from tree-shaking in ESM-compatible bundlers
- Maintain backward compatibility with older Node.js versions

## Module System Architecture

### Build Configuration

The SDK uses TypeScript with a specialized build configuration. The base `tsconfig.json` extends a build configuration and includes both source and test files:

```typescript
{
  "extends": "./tsconfig.build.json",
  "include": ["src/**/*.ts", "test/**/*.ts"],
  "exclude": [],
  "compilerOptions": {
    "noUnusedParameters": true,
    "types": ["@types/bun"],
    "baseUrl": ".",
    "paths": {
      "~test/*": ["./test/*"]
    }
  }
}
```

This configuration supports:
- **Bun runtime** for fast development and testing
- **Path aliases** for cleaner imports in tests
- **Strict TypeScript** checking with unused parameter detection

### Export Strategy

All public APIs are centralized through `src/exports/index.ts`, which serves as the single entry point for the SDK. This approach ensures:

1. **Consistent API surface** across module systems
2. **Explicit export control** - only intended APIs are exposed
3. **Type safety** through TypeScript re-exports

## Module Format Usage

### ES Modules (ESM)

ESM is the modern JavaScript module standard, using `import` and `export` statements:

```typescript
import { createKontorIndexerClient, signet } from '@kontorlabs/kontor';
import type { KontorIndexerClient } from '@kontorlabs/kontor';

const client = createKontorIndexerClient({
  chain: signet,
  transport: http('https://api.signet.kontorx.com')
});
```

**Benefits:**
- Static analysis for better tree-shaking
- Native browser support
- Async module loading
- Better tooling support

### CommonJS (CJS)

CommonJS uses `require()` and `module.exports`:

```javascript
const { createKontorIndexerClient, signet } = require('@kontorlabs/kontor');

const client = createKontorIndexerClient({
  chain: signet,
  transport: require('@kontorlabs/kontor').http('https://api.signet.kontorx.com')
});
```

**Benefits:**
- Wide Node.js compatibility
- Synchronous loading
- Simpler module resolution

## Exported APIs

The SDK exports a comprehensive set of APIs organized by functionality:

### Core Types and Parsing

```typescript
import { 
  type Wit,
  type WitFunction,
  type WitParameter,
  parseWit,
  type ParseWit
} from '@kontorlabs/kontor';
```

### Client Creation

```typescript
import {
  createKontorIndexerClient,
  createKontorWalletClient,
  createBtcPublicClient,
  createBtcWalletClient,
  type KontorIndexerClient,
  type KontorWalletClient
} from '@kontorlabs/kontor';
```

### Account Management

```typescript
import {
  mnemonicToAccount,
  hdKeyToAccount,
  privateKeyToAccount,
  type MnemonicToAccountOptions
} from '@kontorlabs/kontor';
```

### Contract Interactions

```typescript
import {
  viewContract,
  procContract,
  getContractClient,
  type ViewContractParameters,
  type ProcContractParameters
} from '@kontorlabs/kontor';
```

### Encoding and Decoding

```typescript
import {
  encodeFunctionData,
  decodeFunctionResult,
  encodeWitParameters,
  decodeWitParameter,
  type EncodeFunctionDataParameters
} from '@kontorlabs/kontor';
```

## File Extension Conventions

The SDK follows explicit file extension conventions:

### Source Files

All imports in source files use the `.js` extension, even though the source is TypeScript:

```typescript
export { type WitFunction, type Wit, type WitParameter } from "../wit/wit.js";
export { type ParseWit, parseWit } from "../wit/wit-parser/parse-wit.js";
export { nativeToken } from "../sdk/contracts/wits.js";
```

This is because:
- TypeScript outputs JavaScript files with `.js` extensions
- ESM requires explicit file extensions
- The build process compiles `.ts` files to `.js` files

### Type-Only Imports

Type imports are marked with the `type` keyword for optimal tree-shaking:

```typescript
export { type WitFunction } from "../wit/wit.js";
export { type CustomTransport, custom } from "../sdk/clients/transports/custom.js";
```

## Best Practices

### 1. Use Named Imports

Import only what you need for better tree-shaking:

```typescript
// Good
import { createKontorIndexerClient, signet } from '@kontorlabs/kontor';

// Avoid
import * as kontor from '@kontorlabs/kontor';
```

### 2. Separate Type Imports

Use type-only imports when importing types:

```typescript
import { createKontorIndexerClient } from '@kontorlabs/kontor';
import type { KontorIndexerClient } from '@kontorlabs/kontor';
```

### 3. Leverage Transport Abstraction

```typescript
import { http, webSocket, custom } from '@kontorlabs/kontor';
import type { HttpTransport, WebSocketTransport } from '@kontorlabs/kontor';

// HTTP transport
const httpTransport = http('https://api.signet.kontorx.com');

// WebSocket transport
const wsTransport = webSocket('wss://ws.signet.kontorx.com');

// Custom transport
const customTransport = custom({
  request: async ({ method, params }) => {
    // Custom implementation
  }
});
```

## Compatibility Matrix

| Environment | Module System | Support |
|-------------|---------------|----------|
| Node.js 18+ (ESM) | ESM | ✅ Full |
| Node.js 16+ (CJS) | CJS | ✅ Full |
| Modern Bundlers | ESM | ✅ Full |
| Browsers | ESM | ✅ Full |
| Bun | ESM | ✅ Full |

## Troubleshooting

### Module Resolution Issues

If you encounter module resolution errors:

1. **Check package.json exports field** - Ensure your bundler respects the exports map
2. **Verify Node.js version** - Use Node.js 16+ for best compatibility
3. **Enable ESM in Node.js** - Add `"type": "module"` to your package.json or use `.mjs` extensions

### Type Import Errors

For TypeScript type resolution issues:

```typescript
// Ensure moduleResolution is set correctly
{
  "compilerOptions": {
    "moduleResolution": "bundler", // or "node16"
    "esModuleInterop": true
  }
}
```

## Related Resources

- [Client Architecture](./client-architecture) - Understanding client structure
- [Transport System](./transport-system) - Transport layer details
- [Type System](./type-system) - TypeScript type patterns
