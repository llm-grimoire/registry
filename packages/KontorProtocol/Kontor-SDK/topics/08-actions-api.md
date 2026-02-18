---
title: Actions API Reference
slug: actions-api
description: High-level action interfaces for blockchain queries, transactions, and wallet operations
order: 8
category: api
tags: [actions, api, queries, transactions, wallet, public-actions, wallet-actions]
relatedFiles: [src/sdk/actions/index.ts, src/sdk/actions/public/index.ts, src/sdk/actions/wallet/index.ts, src/sdk/client/createPublicClient.ts, src/sdk/client/createWalletClient.ts]
---

# Actions API Reference

## Overview

The Actions API provides high-level interfaces for interacting with blockchain networks. Actions are organized into two main categories: **Public Actions** for read-only blockchain queries and **Wallet Actions** for transaction signing and submission. This abstraction layer simplifies common blockchain operations while maintaining type safety and flexibility.

Actions are designed to be composable and can be extended with custom functionality. They work seamlessly with both public clients (for queries) and wallet clients (for transactions).

## Key Concepts

### Action Architecture

Actions follow a functional design pattern where each action is a standalone function that can be attached to clients:

- **Public Actions**: Read-only operations that query blockchain state
- **Wallet Actions**: Write operations that require signing and sending transactions
- **Composability**: Actions can be combined and extended to create custom workflows
- **Type Safety**: Full TypeScript support with inferred types based on chain configuration

### Client Integration

Actions are typically used through clients rather than called directly:

```typescript
import { createPublicClient, createWalletClient } from './sdk/client'

// Public client with read-only actions
const publicClient = createPublicClient({
  transport: http('https://rpc.example.com')
})

// Wallet client with transaction actions
const walletClient = createWalletClient({
  account: privateKeyToAccount('0x...'),
  transport: http('https://rpc.example.com')
})
```

## Public Actions

Public actions enable read-only interactions with the blockchain. These actions do not require a wallet or signing capabilities.

### Common Public Actions

**Block and Chain Queries**

```typescript
// Get current block number
const blockNumber = await publicClient.getBlockNumber()

// Get block details
const block = await publicClient.getBlock({
  blockNumber: 12345n
})

// Get chain ID
const chainId = await publicClient.getChainId()
```

**Account Queries**

```typescript
// Get account balance
const balance = await publicClient.getBalance({
  address: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb'
})

// Get transaction count (nonce)
const nonce = await publicClient.getTransactionCount({
  address: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb'
})
```

**Transaction Queries**

```typescript
// Get transaction by hash
const transaction = await publicClient.getTransaction({
  hash: '0xabc123...'
})

// Get transaction receipt
const receipt = await publicClient.getTransactionReceipt({
  hash: '0xabc123...'
})

// Wait for transaction confirmation
const receipt = await publicClient.waitForTransactionReceipt({
  hash: '0xabc123...'
})
```

**Contract Queries**

```typescript
// Call contract view function
const result = await publicClient.readContract({
  address: '0xContractAddress',
  abi: contractAbi,
  functionName: 'balanceOf',
  args: ['0xUserAddress']
})

// Get contract bytecode
const bytecode = await publicClient.getBytecode({
  address: '0xContractAddress'
})

// Estimate gas for contract call
const gasEstimate = await publicClient.estimateGas({
  to: '0xContractAddress',
  data: '0x...',
  value: 1000000000000000000n
})
```

**Event Logs**

```typescript
// Get event logs
const logs = await publicClient.getLogs({
  address: '0xContractAddress',
  event: transferEventAbi,
  fromBlock: 12345n,
  toBlock: 12350n
})

// Watch for new events
const unwatch = publicClient.watchEvent({
  address: '0xContractAddress',
  event: transferEventAbi,
  onLogs: (logs) => console.log('New events:', logs)
})
```

## Wallet Actions

Wallet actions enable write operations that modify blockchain state. These actions require a wallet with signing capabilities.

### Transaction Actions

**Sending Transactions**

```typescript
// Send native token transfer
const hash = await walletClient.sendTransaction({
  to: '0xRecipientAddress',
  value: 1000000000000000000n, // 1 ETH in wei
  data: '0x' // optional data
})

// Wait for confirmation
const receipt = await publicClient.waitForTransactionReceipt({ hash })
```

**Contract Interactions**

```typescript
// Write to contract
const hash = await walletClient.writeContract({
  address: '0xContractAddress',
  abi: contractAbi,
  functionName: 'transfer',
  args: ['0xRecipientAddress', 1000000000000000000n]
})

// Deploy contract
const hash = await walletClient.deployContract({
  abi: contractAbi,
  bytecode: '0x...',
  args: ['constructorArg1', 'constructorArg2']
})
```

**Transaction Building**

```typescript
// Prepare transaction request
const request = await walletClient.prepareTransactionRequest({
  to: '0xRecipientAddress',
  value: 1000000000000000000n
})

// Sign transaction
const signedTx = await walletClient.signTransaction(request)

// Send raw signed transaction
const hash = await walletClient.sendRawTransaction({
  serializedTransaction: signedTx
})
```

### Message Signing

```typescript
// Sign message
const signature = await walletClient.signMessage({
  message: 'Hello, blockchain!'
})

// Sign typed data (EIP-712)
const signature = await walletClient.signTypedData({
  domain: {
    name: 'MyDApp',
    version: '1',
    chainId: 1,
    verifyingContract: '0x...'
  },
  types: {
    Person: [
      { name: 'name', type: 'string' },
      { name: 'wallet', type: 'address' }
    ]
  },
  primaryType: 'Person',
  message: {
    name: 'Alice',
    wallet: '0x...'
  }
})
```

## Custom Actions

You can create custom actions to extend client functionality:

```typescript
import { type PublicClient } from './sdk/client'

// Define custom action
export function getTokenBalance(
  client: PublicClient,
  args: { token: `0x${string}`; account: `0x${string}` }
) {
  return client.readContract({
    address: args.token,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [args.account]
  })
}

// Extend client with custom action
const client = createPublicClient({
  transport: http('https://rpc.example.com')
}).extend((client) => ({
  getTokenBalance: (args) => getTokenBalance(client, args)
}))

// Use custom action
const balance = await client.getTokenBalance({
  token: '0xTokenAddress',
  account: '0xUserAddress'
})
```

## Action Composition

Actions can be composed to create complex workflows:

```typescript
// Multi-step transaction workflow
async function transferWithApproval(
  walletClient: WalletClient,
  publicClient: PublicClient,
  token: `0x${string}`,
  spender: `0x${string}`,
  amount: bigint
) {
  // Step 1: Approve spender
  const approvalHash = await walletClient.writeContract({
    address: token,
    abi: erc20Abi,
    functionName: 'approve',
    args: [spender, amount]
  })
  
  // Step 2: Wait for approval confirmation
  await publicClient.waitForTransactionReceipt({ hash: approvalHash })
  
  // Step 3: Execute transfer
  const transferHash = await walletClient.writeContract({
    address: spender,
    abi: contractAbi,
    functionName: 'transferFrom',
    args: [walletClient.account.address, '0xRecipient', amount]
  })
  
  return { approvalHash, transferHash }
}
```

## Error Handling

Actions throw typed errors that can be caught and handled:

```typescript
import { 
  TransactionExecutionError,
  ContractFunctionExecutionError,
  InsufficientFundsError 
} from './sdk/errors'

try {
  const hash = await walletClient.sendTransaction({
    to: '0xRecipientAddress',
    value: 1000000000000000000n
  })
} catch (error) {
  if (error instanceof InsufficientFundsError) {
    console.error('Insufficient funds for transaction')
  } else if (error instanceof TransactionExecutionError) {
    console.error('Transaction failed:', error.message)
  } else {
    console.error('Unexpected error:', error)
  }
}
```

## Best Practices

1. **Use Public Clients for Queries**: Always use public clients for read-only operations to avoid unnecessary wallet connections

2. **Handle Errors Gracefully**: Implement proper error handling for all actions, especially transactions

3. **Estimate Gas Before Sending**: Use `estimateGas` to check transaction viability before submission

4. **Wait for Confirmations**: Use `waitForTransactionReceipt` to ensure transactions are confirmed before proceeding

5. **Type Safety**: Leverage TypeScript types to catch errors at compile time

6. **Action Composition**: Build complex workflows by composing simple actions

## Related Files

For implementation details and advanced usage:

- `src/sdk/actions/index.ts` - Main actions export and organization
- `src/sdk/actions/public/index.ts` - Public action implementations
- `src/sdk/actions/wallet/index.ts` - Wallet action implementations
- `src/sdk/client/createPublicClient.ts` - Public client creation
- `src/sdk/client/createWalletClient.ts` - Wallet client creation
