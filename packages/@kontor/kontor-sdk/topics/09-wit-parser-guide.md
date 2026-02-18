---
title: WIT Parser and ABI Handling
slug: wit-parser-guide
description: Parsing and working with WebAssembly Interface Types for contract ABIs
order: 9
category: architecture
tags: [wit, abi, webassembly, parser, encoding, types]
relatedFiles: [src/wit/wit-parser/index.ts, src/wit/wit-parser/types.ts, src/sdk/utils/wit/encodeWitParameters.ts]
---

# WIT Parser and ABI Handling

## Overview

The WIT (WebAssembly Interface Types) parser is a critical component for working with contract ABIs in the Midnight SDK. It provides the infrastructure for parsing WIT definitions, type handling, and parameter encoding/decoding when interacting with WebAssembly-based smart contracts.

WIT serves as the interface definition language (IDL) for WebAssembly components, defining the types and function signatures that contracts expose. The parser handles the complex task of converting between JavaScript/TypeScript values and the binary representations required by WebAssembly.

## Why WIT for Contract ABIs?

**Type Safety**: WIT provides strongly-typed interface definitions that can be validated at compile-time and runtime.

**WebAssembly Native**: As WebAssembly's standard IDL, WIT integrates naturally with WASM-based contracts.

**Cross-Language Compatibility**: WIT definitions work across different programming languages and toolchains.

**Standardization**: Using WIT aligns with WebAssembly Component Model standards for future compatibility.

## Key Concepts

### WIT Type System

WIT supports a rich type system including:

- **Primitives**: `bool`, `u8`, `u16`, `u32`, `u64`, `s8`, `s16`, `s32`, `s64`, `f32`, `f64`
- **Strings**: `string` for UTF-8 encoded text
- **Compound Types**: `record`, `variant`, `enum`, `flags`, `tuple`
- **Collections**: `list`, `option`
- **Resources**: Opaque handles to contract state

### Parser Architecture

The WIT parser typically consists of:

1. **Lexer/Tokenizer**: Breaks WIT text into tokens
2. **AST Builder**: Constructs an abstract syntax tree from tokens
3. **Type Resolver**: Resolves type references and validates definitions
4. **Code Generator**: Produces TypeScript type definitions and runtime validators

### ABI Encoding

The ABI encoding layer translates between JavaScript values and WebAssembly's linear memory representation:

- **Memory Layout**: Values are packed according to WIT's canonical ABI specification
- **Alignment**: Types are aligned based on their size (u32 at 4-byte boundaries, etc.)
- **String Encoding**: Strings are UTF-8 encoded with length prefixes
- **Complex Types**: Records and variants use tagged unions and discriminators

## Implementation Architecture

### Type Definitions

The type system would define core WIT types:

```typescript
// Core WIT type representations
type WitType = 
  | WitPrimitive
  | WitString
  | WitList
  | WitRecord
  | WitVariant
  | WitOption
  | WitResult
  | WitTuple
  | WitEnum
  | WitFlags;

interface WitRecord {
  kind: 'record';
  name: string;
  fields: Array<{
    name: string;
    type: WitType;
  }>;
}

interface WitVariant {
  kind: 'variant';
  name: string;
  cases: Array<{
    name: string;
    type?: WitType;
  }>;
}

interface WitFunction {
  name: string;
  params: Array<{
    name: string;
    type: WitType;
  }>;
  results: WitType[];
}
```

### Parser Interface

The parser exposes methods for working with WIT definitions:

```typescript
// Example parser API
interface WitParser {
  // Parse WIT text into AST
  parse(source: string): WitWorld;
  
  // Resolve type by name
  resolveType(name: string): WitType | undefined;
  
  // Get function signatures
  getFunctions(): WitFunction[];
  
  // Validate value against type
  validate(value: unknown, type: WitType): boolean;
}

interface WitWorld {
  name: string;
  imports: Map<string, WitInterface>;
  exports: Map<string, WitInterface>;
  types: Map<string, WitType>;
}
```

### Parameter Encoding

Encoding parameters for contract calls involves serializing JavaScript values:

```typescript
// Example encoding implementation
function encodeWitParameters(
  params: unknown[],
  types: WitType[],
  buffer: Uint8Array,
  offset: number = 0
): number {
  let currentOffset = offset;
  
  for (let i = 0; i < params.length; i++) {
    const param = params[i];
    const type = types[i];
    
    currentOffset = encodeValue(param, type, buffer, currentOffset);
  }
  
  return currentOffset;
}

function encodeValue(
  value: unknown,
  type: WitType,
  buffer: Uint8Array,
  offset: number
): number {
  switch (type.kind) {
    case 'u32':
      const view = new DataView(buffer.buffer);
      view.setUint32(offset, value as number, true); // little-endian
      return offset + 4;
      
    case 'string':
      const encoded = new TextEncoder().encode(value as string);
      // Write length prefix
      new DataView(buffer.buffer).setUint32(offset, encoded.length, true);
      // Write string bytes
      buffer.set(encoded, offset + 4);
      return offset + 4 + encoded.length;
      
    case 'record':
      let currentOffset = offset;
      for (const field of type.fields) {
        currentOffset = encodeValue(
          (value as any)[field.name],
          field.type,
          buffer,
          currentOffset
        );
      }
      return currentOffset;
      
    case 'variant':
      // Encode discriminator tag
      const caseIndex = findCaseIndex(value, type);
      new DataView(buffer.buffer).setUint32(offset, caseIndex, true);
      
      // Encode payload if present
      const variantCase = type.cases[caseIndex];
      if (variantCase.type) {
        return encodeValue(
          (value as any).value,
          variantCase.type,
          buffer,
          offset + 4
        );
      }
      return offset + 4;
      
    default:
      throw new Error(`Unsupported type: ${type.kind}`);
  }
}
```

### Parameter Decoding

Decoding reverses the process, reading from binary back to JavaScript:

```typescript
function decodeWitParameters(
  buffer: Uint8Array,
  types: WitType[],
  offset: number = 0
): { values: unknown[]; offset: number } {
  const values: unknown[] = [];
  let currentOffset = offset;
  
  for (const type of types) {
    const result = decodeValue(buffer, type, currentOffset);
    values.push(result.value);
    currentOffset = result.offset;
  }
  
  return { values, offset: currentOffset };
}

function decodeValue(
  buffer: Uint8Array,
  type: WitType,
  offset: number
): { value: unknown; offset: number } {
  const view = new DataView(buffer.buffer);
  
  switch (type.kind) {
    case 'u32':
      return {
        value: view.getUint32(offset, true),
        offset: offset + 4
      };
      
    case 'string':
      const length = view.getUint32(offset, true);
      const bytes = buffer.slice(offset + 4, offset + 4 + length);
      const str = new TextDecoder().decode(bytes);
      return {
        value: str,
        offset: offset + 4 + length
      };
      
    case 'record':
      const record: any = {};
      let currentOffset = offset;
      
      for (const field of type.fields) {
        const result = decodeValue(buffer, field.type, currentOffset);
        record[field.name] = result.value;
        currentOffset = result.offset;
      }
      
      return { value: record, offset: currentOffset };
      
    default:
      throw new Error(`Unsupported type: ${type.kind}`);
  }
}
```

## Usage Patterns

### Parsing Contract ABI

```typescript
import { WitParser } from '@midnight/wit-parser';

const witSource = `
package my-contract;

world contract {
  export transfer: func(to: string, amount: u64) -> result<_, string>;
  export balance-of: func(address: string) -> u64;
}
`;

const parser = new WitParser();
const world = parser.parse(witSource);

// Access exported functions
const transferFn = world.exports.get('transfer');
console.log(transferFn.params); // [{ name: 'to', type: ... }, ...]
```

### Encoding Contract Call Parameters

```typescript
// Prepare parameters for contract call
const params = [
  'midnight1abc123...', // to address
  BigInt(1000000)       // amount
];

const types = transferFn.params.map(p => p.type);
const buffer = new Uint8Array(1024);

const encodedLength = encodeWitParameters(params, types, buffer);
const encodedParams = buffer.slice(0, encodedLength);

// Send to contract
await contract.call('transfer', encodedParams);
```

### Type Validation

```typescript
// Validate parameters before encoding
function validateParams(
  params: unknown[],
  functionSig: WitFunction
): void {
  if (params.length !== functionSig.params.length) {
    throw new Error('Parameter count mismatch');
  }
  
  for (let i = 0; i < params.length; i++) {
    const valid = parser.validate(params[i], functionSig.params[i].type);
    if (!valid) {
      throw new Error(`Invalid parameter ${i}: ${functionSig.params[i].name}`);
    }
  }
}
```

### Generating TypeScript Types

```typescript
// Generate TypeScript definitions from WIT
function generateTypes(world: WitWorld): string {
  let output = '';
  
  // Generate type definitions
  for (const [name, type] of world.types) {
    if (type.kind === 'record') {
      output += `export interface ${name} {\n`;
      for (const field of type.fields) {
        output += `  ${field.name}: ${tsTypeFor(field.type)};\n`;
      }
      output += '}\n\n';
    }
  }
  
  // Generate function signatures
  for (const [name, iface] of world.exports) {
    for (const func of iface.functions) {
      const params = func.params
        .map(p => `${p.name}: ${tsTypeFor(p.type)}`)
        .join(', ');
      const returns = func.results.map(tsTypeFor).join(', ');
      output += `export function ${func.name}(${params}): Promise<${returns}>;\n`;
    }
  }
  
  return output;
}
```

## Integration Points

### Contract Deployment

WIT definitions are embedded in deployed contracts:

```typescript
import { deployContract } from '@midnight/sdk';
import contractWasm from './contract.wasm';
import contractWit from './contract.wit';

const deployment = await deployContract({
  wasm: contractWasm,
  wit: contractWit,
  wallet
});

const contractAddress = deployment.address;
```

### Runtime Type Checking

The parser enables runtime validation:

```typescript
class TypedContract {
  constructor(
    private contract: Contract,
    private wit: WitWorld
  ) {}
  
  async call(method: string, ...args: unknown[]): Promise<unknown> {
    const func = this.wit.exports.get(method);
    if (!func) {
      throw new Error(`Method ${method} not found`);
    }
    
    // Validate arguments
    validateParams(args, func);
    
    // Encode and call
    const encoded = encodeWitParameters(
      args,
      func.params.map(p => p.type),
      new Uint8Array(4096)
    );
    
    const result = await this.contract.call(method, encoded);
    
    // Decode result
    return decodeWitParameters(
      result,
      func.results
    ).values[0];
  }
}
```

## Best Practices

1. **Cache Parsed WIT**: Parse WIT definitions once and reuse the parsed representation
2. **Validate Early**: Check parameter types before encoding to fail fast
3. **Buffer Management**: Pre-allocate buffers for encoding to avoid reallocations
4. **Error Handling**: Provide clear error messages for type mismatches and encoding failures
5. **Version Compatibility**: Check WIT version compatibility between SDK and contracts

## Related Components

- **Contract SDK**: Uses WIT parser for contract interactions
- **TypeScript Generator**: Generates type-safe contract clients from WIT
- **WASM Runtime**: Executes contracts defined by WIT interfaces
- **State Management**: Serializes state using WIT-defined types

## Further Reading

- WebAssembly Component Model specification
- WIT language reference
- Canonical ABI specification
- Contract development guide
