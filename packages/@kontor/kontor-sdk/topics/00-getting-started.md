---
title: Getting Started with Kontor SDK
slug: getting-started
description: Installation, basic setup, and first contract interaction walkthrough
order: 0
category: guides
tags: [quickstart, installation, setup, beginner, tutorial]
relatedFiles: [src/exports/index.ts, src/sdk/clients/kontor/create-public-client.js, src/sdk/clients/kontor/create-wallet-client.js, src/sdk/accounts/private-key-to-account.js, src/sdk/actions/kontor/public/view-contract.js, src/sdk/actions/kontor/public/proc-contract.js]
---

# Getting Started with Kontor SDK

The Kontor SDK provides a TypeScript interface for interacting with Bitcoin smart contracts on the Kontor network. This guide will walk you through installation, basic setup, and your first contract interaction.

## Overview

Kontor SDK enables developers to:
- Connect to Kontor blockchain networks
- Create and manage Bitcoin accounts
- Read contract state (view functions)
- Execute contract transactions (proc functions)
- Compose and broadcast Bitcoin transactions

The SDK follows a client-based architecture with two primary client types:
- **KontorIndexerClient** - For read-only operations (viewing contract state)
- **KontorWalletClient** - For transaction signing and submission

## Installation

Install the Kontor SDK via npm:

```bash
npm install @kontor/sdk
```

Or using yarn:

```bash
yarn add @kontor/sdk
```

## Basic Setup

### Creating a Public Client

The `KontorIndexerClient` allows you to read blockchain data and view contract state without signing transactions:

```typescript
import { createKontorIndexerClient, http, signet } from '@kontor/sdk';

const publicClient = createKontorIndexerClient({
  chain: signet,
  transport: http('https://signet-indexer.kontor.network')
});
```

### Creating a Wallet Client

The `KontorWalletClient` enables transaction signing and submission:

```typescript
import { 
  createKontorWalletClient, 
  http, 
  signet,
  privateKeyToAccount 
} from '@kontor/sdk';

// Create an account from a private key
const account = privateKeyToAccount({
  privateKey: '0x...' // Your private key
});

const walletClient = createKontorWalletClient({
  account,
  chain: signet,
  transport: http('https://signet-indexer.kontor.network')
});
```

## Account Management

Kontor SDK supports multiple account creation methods:

### From Private Key

```typescript
import { privateKeyToAccount } from '@kontor/sdk';

const account = privateKeyToAccount({
  privateKey: '0x1234567890abcdef...'
});
```

### From Mnemonic

```typescript
import { mnemonicToAccount } from '@kontor/sdk';

const account = mnemonicToAccount({
  mnemonic: 'your twelve word mnemonic phrase here...',
  accountIndex: 0 // Optional: defaults to 0
});
```

### From HD Key

```typescript
import { hdKeyToAccount } from '@kontor/sdk';

const account = hdKeyToAccount({
  hdKey: hdKeyInstance,
  accountIndex: 0
});
```

## First Contract Interaction

### Reading Contract State (View)

View functions read contract state without modifying it or requiring gas:

```typescript
import { viewContract } from '@kontor/sdk';

// Read a contract's state
const result = await publicClient.viewContract({
  contractAddress: 'tb1q...', // Contract address
  wit: contractWit, // Contract WIT definition
  functionName: 'getBalance',
  args: ['tb1q...'] // Function arguments
});

console.log('Balance:', result);
```

### Executing Contract Transactions (Proc)

Proc functions modify contract state and require transaction fees:

```typescript
import { procContract } from '@kontor/sdk';

// Execute a state-changing function
const txHash = await walletClient.procContract({
  contractAddress: 'tb1q...', // Contract address
  wit: contractWit, // Contract WIT definition
  functionName: 'transfer',
  args: ['tb1q...', 1000n], // Recipient and amount
  feeRate: 10 // Satoshis per vbyte
});

console.log('Transaction hash:', txHash);
```

## Working with Contract Clients

For a more ergonomic interface, use the contract client pattern:

```typescript
import { getContractClient } from '@kontor/sdk';

// Create a typed contract client
const contractClient = getContractClient({
  address: 'tb1q...',
  wit: contractWit,
  client: publicClient
});

// Call view functions directly
const balance = await contractClient.read.getBalance(['tb1q...']);

// For wallet operations, bind the wallet client
const writeContract = getContractClient({
  address: 'tb1q...',
  wit: contractWit,
  client: walletClient
});

await writeContract.write.transfer(['tb1q...', 1000n]);
```

## Transport Configuration

Kontor SDK supports multiple transport types:

### HTTP Transport

```typescript
import { http } from '@kontor/sdk';

const transport = http('https://signet-indexer.kontor.network', {
  timeout: 30000, // Optional: request timeout in ms
  fetchOptions: {} // Optional: custom fetch options
});
```

### WebSocket Transport

```typescript
import { webSocket } from '@kontor/sdk';

const transport = webSocket('wss://signet-indexer.kontor.network');
```

### Custom Transport

```typescript
import { custom } from '@kontor/sdk';

const transport = custom({
  async request({ method, params }) {
    // Implement custom transport logic
    return response;
  }
});
```

## Working with WIT Definitions

WIT (Web Interface Types) defines your contract's interface. Parse WIT definitions to use with the SDK:

```typescript
import { parseWit } from '@kontor/sdk';

const witSource = `
interface token {
  func transfer(to: address, amount: u64) -> bool;
  func balance-of(owner: address) -> u64;
}
`;

const wit = parseWit(witSource);
```

## Complete Example

Here's a complete example bringing everything together:

```typescript
import {
  createKontorIndexerClient,
  createKontorWalletClient,
  privateKeyToAccount,
  http,
  signet,
  parseWit,
  getContractClient
} from '@kontor/sdk';

// Setup
const account = privateKeyToAccount({
  privateKey: process.env.PRIVATE_KEY
});

const publicClient = createKontorIndexerClient({
  chain: signet,
  transport: http('https://signet-indexer.kontor.network')
});

const walletClient = createKontorWalletClient({
  account,
  chain: signet,
  transport: http('https://signet-indexer.kontor.network')
});

// Parse contract WIT
const wit = parseWit(`
  interface token {
    func balance-of(owner: address) -> u64;
    func transfer(to: address, amount: u64) -> bool;
  }
`);

// Create contract client
const contract = getContractClient({
  address: 'tb1qcontractaddress...',
  wit,
  client: walletClient
});

// Read balance
const balance = await contract.read.balanceOf([account.address]);
console.log('My balance:', balance);

// Transfer tokens
const txHash = await contract.write.transfer([
  'tb1qrecipient...',
  100n
]);
console.log('Transfer transaction:', txHash);
```

## Next Steps

- Explore [Contract Interaction Patterns](./contract-patterns) for advanced usage
- Learn about [WIT Definitions](./wit-definitions) to understand contract interfaces
- Review [Transaction Composition](./transaction-composition) for custom transaction building
- Check [Account Management](./account-management) for advanced account features

## Common Patterns

### Error Handling

```typescript
try {
  const result = await publicClient.viewContract({
    contractAddress: 'tb1q...',
    wit: contractWit,
    functionName: 'getBalance',
    args: ['tb1q...']
  });
} catch (error) {
  console.error('Contract call failed:', error);
}
```

### Environment Configuration

```typescript
const config = {
  chain: signet,
  transport: http(process.env.KONTOR_RPC_URL || 'https://signet-indexer.kontor.network'),
  account: privateKeyToAccount({
    privateKey: process.env.PRIVATE_KEY
  })
};
```

This guide provides the foundation for building with Kontor SDK. The modular architecture allows you to compose clients, accounts, and transports to fit your application's needs.
