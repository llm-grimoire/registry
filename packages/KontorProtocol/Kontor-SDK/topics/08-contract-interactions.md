---
title: Contract Interactions and WIT Specifications
slug: contract-interactions
description: Type-safe contract method invocation using WebAssembly Interface Types
order: 8
category: architecture
tags: [contracts, wit, type-safety, webassembly, type-system]
relatedFiles: [src/sdk/types/contract.ts, src/wit/wit-parser/index.ts, src/wit/wit.js, src/wit/register.js, src/wit/utils.js, src/wit/type-utils.js]
---

# Contract Interactions and WIT Specifications

## Overview

The contract interaction system provides type-safe method invocation for smart contracts using WebAssembly Interface Types (WIT) specifications. This architecture ensures compile-time type safety when calling contract methods, automatically validating function names, parameters, and return types based on the contract's WIT definition.

WIT specifications serve as the type schema for contracts, defining the interface between your application and deployed smart contracts. The type system extracts and transforms WIT definitions into TypeScript types, enabling full IDE support and catching errors before runtime.

## Key Concepts

### WebAssembly Interface Types (WIT)

WIT is a schema definition language that describes the public interface of WebAssembly modules. In this context, WIT specifications define:

- **Function signatures**: Method names, parameters, and return types
- **State mutability**: Whether functions are read-only (`view`) or state-changing (`write`)
- **Type definitions**: Custom types used across the contract interface

The WIT specification acts as a contract between your frontend code and the smart contract, ensuring both sides agree on the interface.

### Type-Safe Contract Function Names

The `ContractFunctionName` type extracts valid function names from a WIT specification based on the desired state mutability:

```typescript
export type ContractFunctionName<
  wit extends Wit | readonly unknown[] = Wit,
  mutability extends WitStateMutability = WitStateMutability,
> =
  ExtractWitFunctionNames<
    wit extends Wit ? wit : Wit,
    mutability
  > extends infer functionName extends string
    ? [functionName] extends [never]
      ? string
      : functionName
    : string;
```

This type:
- Accepts a WIT specification and optional mutability constraint
- Extracts function names matching the mutability requirement
- Falls back to `string` if no specific functions are found
- Provides autocomplete for valid function names in your IDE

### Type-Safe Function Arguments

The `ContractFunctionArgs` type derives the expected argument types for a specific contract function:

```typescript
export type ContractFunctionArgs<
  wit extends Wit | readonly unknown[] = Wit,
  mutability extends WitStateMutability = WitStateMutability,
  functionName extends ContractFunctionName<
    wit,
    mutability
  > = ContractFunctionName<wit, mutability>,
> =
  WitParametersToPrimitiveTypes<
    ExtractWitFunction<
      wit extends Wit ? wit : Wit,
      functionName,
      mutability
    >["inputs"],
    "inputs"
  > extends infer args
    ? [args] extends [never]
      ? readonly unknown[]
      : args
    : readonly unknown[];
```

This transforms WIT parameter definitions into TypeScript tuple types, converting WIT types to their TypeScript equivalents.

### Type Widening for Runtime Flexibility

The `Widen` type provides controlled type widening to handle runtime values while maintaining type safety:

```typescript
export type Widen<T> =
  // preserve unknown / any
  | ([unknown] extends [T] ? unknown : never)
  // preserve functions
  | (T extends (...args: any[]) => any ? T : never)
  // bigint-ish -> bigint
  | (T extends ResolvedRegister["bigIntType"] ? bigint : never)
  // booleans -> boolean
  | (T extends boolean ? boolean : never)
  // int-ish -> number
  | (T extends ResolvedRegister["intType"] ? number : never)
  // decimal -> canonical tuple or number (NO string)
  | (T extends ResolvedRegister["decimalType"]
      ? ResolvedRegister["decimalType"]
      : never)
  // addresses -> registered address type
  | (T extends ResolvedRegister["addressType"]
      ? ResolvedRegister["addressType"]
      : never)
  // plain strings (non-address)
  | (T extends string
      ? T extends ResolvedRegister["addressType"]
        ? never
        : string
      : never)
  // empty tuple
  | (T extends readonly [] ? readonly [] : never)
  // object / record types (but not arrays/tuples)
  | (T extends readonly any[]
      ? never
      : T extends Record<string, unknown>
        ? { [K in keyof T]: Widen<T[K]> }
        : never)
  // tuple/array-like types: recursively widen elements
  | (T extends { length: number }
      ? {
          [K in keyof T]: Widen<T[K]>;
        } extends infer V extends readonly unknown[]
        ? readonly [...V]
        : never
      : never);
```

This type:
- Preserves function types and unknown/any
- Widens specific numeric types (bigint, number) to their primitive forms
- Handles registered types (addresses, decimals) through the type registry
- Recursively widens nested objects and tuples
- Excludes string widening for address types to maintain type safety

### Contract Function Parameters

The `ContractFunctionParameters` type combines all necessary information for a contract call:

```typescript
export type ContractFunctionParameters<
  wit extends Wit | readonly unknown[] = Wit,
  mutability extends WitStateMutability = WitStateMutability,
  functionName extends ContractFunctionName<
    wit,
    mutability
  > = ContractFunctionName<wit, mutability>,
  args extends ContractFunctionArgs<
    wit,
    mutability,
    functionName
  > = ContractFunctionArgs<wit, mutability, functionName>,
  allFunctionNames = ContractFunctionName<wit, mutability>,
  allArgs = ContractFunctionArgs<wit, mutability, functionName>,
> = {
  wit: wit;
  functionName:
    | allFunctionNames
    | (functionName extends allFunctionNames ? functionName : never);
  contractAddress: ResolvedRegister["contractAddress"];
} & (readonly [] extends allArgs
  ? { args?: allArgs | undefined }
  : { args: args });
```

This type:
- Requires the WIT specification for type derivation
- Ensures function name is valid for the contract
- Requires the contract address
- Makes `args` optional when the function takes no parameters
- Makes `args` required when parameters are expected

### Contract Function Return Types

The `ContractFunctionReturnType` type derives the expected return type:

```typescript
export type ContractFunctionReturnType<
  wit extends Wit | readonly unknown[] = Wit,
  mutability extends WitStateMutability = WitStateMutability,
  functionName extends ContractFunctionName<
    wit,
    mutability
  > = ContractFunctionName<wit, mutability>,
  args extends ContractFunctionArgs<
    wit,
    mutability,
    functionName
  > = ContractFunctionArgs<wit, mutability, functionName>,
> = wit extends Wit
  ? Wit extends wit
    ? unknown
    : WitParametersToPrimitiveTypes<
          ExtractWitFunctionForArgs<
            wit,
            mutability,
            functionName,
            args
          >["outputs"]
        > extends infer types
      ? types extends readonly []
        ? void
        : types extends readonly [infer type]
          ? type
          : types
      : never
  : unknown;
```

This type:
- Returns `void` for functions with no outputs
- Unwraps single-element return tuples to the single type
- Returns tuple types for multiple outputs
- Falls back to `unknown` for untyped WIT specifications

### Function Overload Resolution

The `ExtractWitFunctionForArgs` type handles function overloading by matching arguments:

```typescript
export type ExtractWitFunctionForArgs<
  wit extends Wit,
  mutability extends WitStateMutability,
  functionName extends ContractFunctionName<wit, mutability>,
  args extends ContractFunctionArgs<wit, mutability, functionName>,
> =
  ExtractWitFunction<
    wit,
    functionName,
    mutability
  > extends infer witFunction extends WitFunction
    ? IsUnion<witFunction> extends true
      ? UnionToTuple<witFunction> extends infer witFunctions extends
          readonly WitFunction[]
```

This type:
- Detects when multiple function overloads exist (union types)
- Converts overload unions to tuples for filtering
- Selects the matching overload based on provided arguments
- Ensures type safety even with polymorphic functions

## Type System Architecture

### Registered Types Integration

The contract type system integrates with the global type registry through `ResolvedRegister`:

- **`contractAddress`**: The registered contract address type
- **`addressType`**: General address type for parameters
- **`bigIntType`**: Large integer type handling
- **`intType`**: Standard integer type
- **`decimalType`**: Decimal number representation

These registered types ensure consistency across the entire application and allow customization of type representations.

### WIT Parameter Transformation

The `WitParametersToPrimitiveTypes` utility (referenced but not shown) transforms WIT parameter definitions into TypeScript types:

1. Parses WIT parameter specifications
2. Maps WIT primitive types to TypeScript primitives
3. Handles complex types (records, variants, lists)
4. Preserves type names and optional parameters
5. Generates tuple types maintaining parameter order

### State Mutability Filtering

Functions are categorized by state mutability:

- **`view`**: Read-only functions that don't modify contract state
- **`write`**: State-changing functions that modify contract storage

The type system uses mutability to filter available functions and apply appropriate type constraints.

## Usage Patterns

### Basic Contract Interaction

```typescript
import type { ContractFunctionParameters, ContractFunctionReturnType } from './sdk/types/contract';
import type { MyContractWit } from './contracts/my-contract.wit';

// Type-safe function call parameters
type TransferParams = ContractFunctionParameters<
  MyContractWit,
  'write',
  'transfer'
>;

// Expected return type
type TransferResult = ContractFunctionReturnType<
  MyContractWit,
  'write',
  'transfer'
>;

const params: TransferParams = {
  wit: myContractWit,
  functionName: 'transfer', // Autocomplete available
  contractAddress: '0x123...',
  args: [recipient, amount] // Type-checked tuple
};
```

### View Function with No Arguments

```typescript
type GetBalanceParams = ContractFunctionParameters<
  MyContractWit,
  'view',
  'getBalance'
>;

const params: GetBalanceParams = {
  wit: myContractWit,
  functionName: 'getBalance',
  contractAddress: '0x123...',
  // args is optional when function takes no parameters
};
```

### Handling Multiple Return Values

```typescript
// Function returning multiple values
type MultiReturnResult = ContractFunctionReturnType<
  MyContractWit,
  'view',
  'getAccountInfo'
>;
// Result type: [balance: bigint, nonce: number, isActive: boolean]

const [balance, nonce, isActive] = await contract.call(params);
```

### Generic Contract Interaction

```typescript
function callContract<
  TWit extends Wit,
  TMutability extends WitStateMutability,
  TFunctionName extends ContractFunctionName<TWit, TMutability>
>(
  params: ContractFunctionParameters<TWit, TMutability, TFunctionName>
): Promise<ContractFunctionReturnType<TWit, TMutability, TFunctionName>> {
  // Implementation with full type safety
}
```

## Best Practices

### Always Use WIT Specifications

Define WIT specifications for all contracts to enable type safety:

```typescript
// Define WIT specification
const tokenWit = {
  functions: [
    {
      name: 'transfer',
      inputs: [{ name: 'to', type: 'address' }, { name: 'amount', type: 'u256' }],
      outputs: [{ name: 'success', type: 'bool' }],
      mutability: 'write'
    }
  ]
} as const satisfies Wit;
```

### Leverage Type Narrowing

Use specific mutability types to restrict available functions:

```typescript
// Only allow read-only functions
type ViewFunctions = ContractFunctionName<MyContractWit, 'view'>;

// Only allow state-changing functions
type WriteFunctions = ContractFunctionName<MyContractWit, 'write'>;
```

### Handle Type Widening Appropriately

Use `Widen` when accepting user input or runtime values:

```typescript
import type { Widen, UnionWiden } from './sdk/types/contract';

// Accept wider types for user input
function submitTransaction(
  args: Widen<ContractFunctionArgs<MyContractWit, 'write', 'transfer'>>
) {
  // Implementation
}
```

## Related Type Utilities

The contract interaction system relies on several utility types:

- **`ExtractWitFunction`**: Extracts specific function definitions from WIT
- **`ExtractWitFunctionNames`**: Gets all function names matching criteria
- **`WitParametersToPrimitiveTypes`**: Converts WIT types to TypeScript
- **`IsUnion`**: Detects union types for overload handling
- **`UnionToTuple`**: Converts union types to tuple types

These utilities work together to provide comprehensive type safety throughout the contract interaction lifecycle.

## Type Safety Benefits

1. **Compile-time validation**: Catch invalid function names and arguments before runtime
2. **IDE autocomplete**: Full IntelliSense support for function names and parameters
3. **Refactoring safety**: Changes to WIT specifications propagate through the type system
4. **Documentation**: Types serve as inline documentation of contract interfaces
5. **Version compatibility**: Type mismatches reveal contract version incompatibilities

## Related Files

For deeper understanding of the contract interaction system:

- **`src/wit/wit.js`**: Core WIT type definitions and interfaces
- **`src/wit/register.js`**: Type registry for customizable type mappings
- **`src/wit/utils.js`**: WIT extraction and transformation utilities
- **`src/wit/type-utils.js`**: Advanced type manipulation helpers
- **`src/wit/wit-parser/index.ts`**: WIT specification parsing logic
