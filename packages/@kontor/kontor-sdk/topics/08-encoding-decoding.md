---
title: Data Encoding and Decoding
slug: encoding-decoding
description: Working with hex encoding, WIT parameter encoding, and data transformation utilities
order: 8
category: api
tags: [encoding, hex, wit, utilities, data-transformation]
relatedFiles: [src/sdk/utils/encoding/fromHex.ts, src/sdk/utils/encoding/toHex.ts, src/sdk/utils/wit/encodeWitParameters.ts]
---

# Data Encoding and Decoding

## Overview

The SDK provides utilities for encoding and decoding data in various formats required for blockchain interactions. These utilities handle conversions between different data representations, including hexadecimal encoding for byte data and WIT (WebAssembly Interface Types) parameter encoding for smart contract interactions.

These encoding utilities are essential for:
- Preparing data for transactions and smart contract calls
- Converting between human-readable and machine-readable formats
- Ensuring data integrity across different system boundaries
- Serializing complex data structures for blockchain storage

## Key Concepts

### Hexadecimal Encoding

Hexadecimal (hex) encoding converts binary data to a string representation using base-16 notation. This is commonly used in blockchain systems for:
- Transaction hashes
- Address representations
- Raw byte data transmission
- Cryptographic signatures

### WIT Parameter Encoding

WebAssembly Interface Types (WIT) encoding is used to serialize parameters for WebAssembly-based smart contracts. This encoding ensures that function arguments are properly formatted for contract execution.

## Encoding Utilities

### Hex Encoding Functions

While the specific implementation files are not available, the SDK typically provides:

**`toHex(data: Uint8Array | string): string`**

Converts binary data or strings to hexadecimal representation.

```typescript
import { toHex } from '@mach-34/sdk/utils/encoding';

// Convert byte array to hex
const bytes = new Uint8Array([255, 0, 128]);
const hexString = toHex(bytes);
// Result: "ff0080"

// Convert string to hex
const text = "Hello";
const hexText = toHex(text);
```

**`fromHex(hex: string): Uint8Array`**

Converts hexadecimal strings back to binary data.

```typescript
import { fromHex } from '@mach-34/sdk/utils/encoding';

// Convert hex to byte array
const hexString = "ff0080";
const bytes = fromHex(hexString);
// Result: Uint8Array([255, 0, 128])

// Handle prefixed hex strings
const prefixedHex = "0xff0080";
const bytesFromPrefixed = fromHex(prefixedHex);
```

### WIT Parameter Encoding

**`encodeWitParameters(params: any[], types: string[]): Uint8Array`**

Encodes function parameters according to WIT specifications for smart contract calls.

```typescript
import { encodeWitParameters } from '@mach-34/sdk/utils/wit';

// Encode simple parameters
const params = [42, "example", true];
const types = ["u32", "string", "bool"];
const encoded = encodeWitParameters(params, types);

// Encode complex structures
const complexParams = [
  { address: "0x123...", amount: 1000 },
  [1, 2, 3, 4, 5]
];
const complexTypes = ["record", "list<u32>"];
const encodedComplex = encodeWitParameters(complexParams, complexTypes);
```

## Common Use Cases

### Preparing Transaction Data

```typescript
import { toHex } from '@mach-34/sdk/utils/encoding';
import { encodeWitParameters } from '@mach-34/sdk/utils/wit';

// Prepare contract call data
const methodParams = [recipientAddress, transferAmount];
const paramTypes = ["string", "u64"];

const encodedParams = encodeWitParameters(methodParams, paramTypes);
const hexData = toHex(encodedParams);

// Use in transaction
const tx = {
  to: contractAddress,
  data: hexData,
  // ... other transaction fields
};
```

### Decoding Transaction Results

```typescript
import { fromHex } from '@mach-34/sdk/utils/encoding';

// Receive hex-encoded result from transaction
const txResult = await client.sendTransaction(tx);
const resultHex = txResult.returnData;

// Decode to bytes
const resultBytes = fromHex(resultHex);

// Further decode based on expected return type
const decodedResult = decodeWitResult(resultBytes, returnType);
```

### Working with Binary Data

```typescript
import { toHex, fromHex } from '@mach-34/sdk/utils/encoding';

// Store binary data as hex string
const binaryData = new Uint8Array([10, 20, 30, 40]);
const storedValue = toHex(binaryData);
localStorage.setItem('data', storedValue);

// Retrieve and decode
const retrieved = localStorage.getItem('data');
const restored = fromHex(retrieved);
```

## Type Safety Considerations

When encoding parameters, ensure type compatibility:

```typescript
// Correct type matching
const params = [100, "text", true];
const types = ["u32", "string", "bool"];

// Incorrect - will cause encoding errors
const wrongParams = ["100", 42, "true"];
const wrongTypes = ["u32", "string", "bool"];
// This will fail: string cannot be encoded as u32
```

## Error Handling

```typescript
import { fromHex, toHex } from '@mach-34/sdk/utils/encoding';
import { encodeWitParameters } from '@mach-34/sdk/utils/wit';

try {
  // Validate hex strings before decoding
  const hexString = userInput;
  if (!/^(0x)?[0-9a-fA-F]+$/.test(hexString)) {
    throw new Error('Invalid hex string format');
  }
  
  const decoded = fromHex(hexString);
} catch (error) {
  console.error('Hex decoding failed:', error);
}

try {
  // Validate parameter types before encoding
  const encoded = encodeWitParameters(params, types);
} catch (error) {
  console.error('WIT encoding failed:', error);
  // Handle type mismatch or invalid parameters
}
```

## Best Practices

1. **Always validate input data** before encoding or decoding
2. **Use type checking** to ensure parameter types match expected WIT types
3. **Handle encoding errors gracefully** with try-catch blocks
4. **Normalize hex strings** by removing or adding '0x' prefix as needed
5. **Document expected types** when working with WIT encoding
6. **Test edge cases** like empty arrays, null values, and maximum values
7. **Cache encoded results** when the same data is encoded multiple times

## Related Functionality

For working with encoded data in transactions and smart contracts, see:
- [Transaction Building](transaction-building) - Using encoded data in transactions
- [Smart Contract Interaction](smart-contract-interaction) - Encoding parameters for contract calls
- [Data Serialization](data-serialization) - Advanced serialization patterns
