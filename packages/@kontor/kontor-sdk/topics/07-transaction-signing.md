---
title: Transaction Signing with PSBT
slug: transaction-signing
description: Creating and signing Partially Signed Bitcoin Transactions for contract interactions
order: 7
category: guides
tags: [psbt, transactions, signing, bitcoin, wallet]
relatedFiles: [src/sdk/actions/wallet/signPsbt.ts, src/sdk/accounts/utils/signPsbt.ts, src/sdk/types/psbt.ts]
---

# Transaction Signing with PSBT

## Overview

Partially Signed Bitcoin Transactions (PSBT) are a critical component for creating, signing, and broadcasting Bitcoin transactions in sCrypt applications. PSBTs enable multi-party transaction workflows and allow different components to contribute to transaction construction and signing without requiring complete transaction information upfront.

This guide covers how to work with PSBTs in the sCrypt SDK for signing transactions that interact with smart contracts on Bitcoin.

## Why Use PSBT?

PSBT provides several key advantages:

- **Separation of Concerns**: Transaction construction can be separated from signing
- **Multi-Signature Support**: Multiple parties can sign the same transaction independently
- **Hardware Wallet Compatibility**: Standard format supported by hardware wallets
- **Incomplete Information**: Can be created without all inputs/outputs finalized
- **Interoperability**: Standard format (BIP 174) ensures compatibility across Bitcoin tools

## Key Concepts

### PSBT Structure

A PSBT contains:

- **Global Data**: Transaction version, inputs, outputs
- **Input Data**: Previous transaction information, signatures, scripts
- **Output Data**: Redeem scripts, derivation paths
- **Metadata**: Additional information for signing

### PSBT Lifecycle

1. **Creator**: Initializes the PSBT with transaction structure
2. **Updater**: Adds input/output information
3. **Signer**: Signs inputs with private keys
4. **Combiner**: Merges multiple PSBTs (for multi-sig)
5. **Finalizer**: Converts signed PSBT to complete transaction
6. **Extractor**: Broadcasts the finalized transaction

## PSBT Signing in sCrypt SDK

### Basic PSBT Signing Workflow

When interacting with smart contracts, you typically:

1. Build a transaction that calls a contract method
2. Create a PSBT from the transaction
3. Sign the PSBT with your wallet
4. Finalize and broadcast the signed transaction

### Signing PSBTs

The sCrypt SDK provides utilities for signing PSBTs through wallet integrations. While the specific implementation files are not available, the typical pattern involves:

```typescript
// Typical PSBT signing pattern
import { signPsbt } from './sdk/actions/wallet/signPsbt';

// Create a PSBT from your transaction
const psbt = transaction.toPSBT();

// Sign the PSBT with your wallet
const signedPsbt = await signPsbt(psbt, {
  autoFinalize: true,
  signingIndexes: [0, 1], // Which inputs to sign
});

// Extract and broadcast
const signedTx = signedPsbt.extractTransaction();
await signedTx.broadcast();
```

### Account-Level Signing

Account utilities provide lower-level PSBT signing capabilities:

```typescript
import { signPsbt } from './sdk/accounts/utils/signPsbt';

// Sign with specific account/key
const signature = await signPsbt(psbt, account, inputIndex);
```

## PSBT Type Definitions

The SDK includes TypeScript types for PSBT operations:

```typescript
// Example type structures for PSBT
interface PSBTInput {
  witnessUtxo?: {
    script: Buffer;
    value: number;
  };
  nonWitnessUtxo?: Buffer;
  sighashType?: number;
  // Additional fields
}

interface PSBTOutput {
  script: Buffer;
  value: number;
  // Additional fields
}

interface PSBT {
  inputs: PSBTInput[];
  outputs: PSBTOutput[];
  signInput(index: number, keyPair: any): void;
  finalizeAllInputs(): void;
  extractTransaction(): Transaction;
}
```

## Working with Contract Interactions

### Signing Contract Method Calls

When calling smart contract methods:

```typescript
// Build the contract call transaction
const { tx } = await contract.methods.unlock(
  arg1,
  arg2,
  {
    pubKeyOrAddrToSign: myPublicKey,
  }
);

// The transaction is typically auto-signed by the SDK
// But you can also work with PSBTs explicitly:
const psbt = tx.toPSBT();
const signedPsbt = await wallet.signPsbt(psbt);
```

### Multi-Input Transactions

For transactions with multiple inputs requiring different signatures:

```typescript
const psbt = new PSBT();

// Add inputs from different sources
psbt.addInput(input1);
psbt.addInput(input2);

// Sign specific inputs
await signPsbt(psbt, { signingIndexes: [0] }); // Sign first input
await signPsbt(psbt, { signingIndexes: [1] }); // Sign second input

// Finalize once all signatures are collected
psbt.finalizeAllInputs();
```

## Best Practices

### Verify Before Signing

Always verify transaction details before signing:

```typescript
// Check outputs and amounts
const outputs = psbt.txOutputs;
for (const output of outputs) {
  console.log(`Sending ${output.value} satoshis to ${output.address}`);
}

// Verify you're signing the expected inputs
const inputsToSign = psbt.data.inputs.filter((_, i) => 
  signingIndexes.includes(i)
);
```

### Handle Errors Gracefully

```typescript
try {
  const signedPsbt = await signPsbt(psbt);
} catch (error) {
  if (error.message.includes('User rejected')) {
    // Handle user cancellation
  } else if (error.message.includes('Insufficient funds')) {
    // Handle insufficient balance
  } else {
    // Handle other signing errors
  }
}
```

### Finalization

Ensure PSBTs are properly finalized before extraction:

```typescript
// Check if PSBT is fully signed
if (psbt.validateSignaturesOfAllInputs()) {
  psbt.finalizeAllInputs();
  const tx = psbt.extractTransaction();
  await tx.broadcast();
} else {
  throw new Error('PSBT not fully signed');
}
```

## Integration with Wallets

### Browser Wallets

Most Bitcoin browser wallets support PSBT signing:

```typescript
// Request wallet to sign PSBT
const signedPsbtHex = await window.unisat.signPsbt(psbtHex);
const signedPsbt = PSBT.fromHex(signedPsbtHex);
```

### Hardware Wallets

PSBTs enable secure hardware wallet integration:

```typescript
// Export PSBT for hardware wallet
const psbtBase64 = psbt.toBase64();

// User signs with hardware wallet externally
// Import signed PSBT
const signedPsbt = PSBT.fromBase64(signedPsbtBase64);
```

## Common Patterns

### Batch Signing

Sign multiple PSBTs in a single user interaction:

```typescript
const psbts = [psbt1, psbt2, psbt3];
const signedPsbts = await Promise.all(
  psbts.map(psbt => signPsbt(psbt))
);
```

### Conditional Signing

Sign only certain inputs based on conditions:

```typescript
const signingIndexes = psbt.data.inputs
  .map((input, index) => ({ input, index }))
  .filter(({ input }) => shouldSignInput(input))
  .map(({ index }) => index);

await signPsbt(psbt, { signingIndexes });
```

## Related Resources

- [Wallet Integration](./wallet-integration) - Connecting to Bitcoin wallets
- [Transaction Building](./transaction-building) - Constructing Bitcoin transactions
- [Smart Contract Methods](./contract-methods) - Calling contract methods
- [BIP 174 Specification](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) - PSBT standard

## Next Steps

- Explore wallet integration patterns for automatic PSBT handling
- Learn about transaction fee estimation and management
- Understand contract interaction patterns requiring signatures
- Study multi-signature and time-locked transaction patterns
