---
title: Account Management and Key Handling
slug: account-management
description: Working with HD keys, mnemonics, private keys, and P2TR address generation
order: 3
category: guides
tags: [accounts, keys, mnemonic, hd-wallet, p2tr, bitcoin, security]
relatedFiles: [src/sdk/accounts/index.ts, src/sdk/accounts/hdKey.ts, src/sdk/accounts/mnemonic.ts, src/sdk/accounts/privateKey.ts]
---

## Overview

Account management is fundamental to working with Bitcoin and blockchain applications. This guide covers the key concepts and implementations for handling hierarchical deterministic (HD) wallets, mnemonic phrases, private keys, and Pay-to-Taproot (P2TR) address generation.

The account management system provides secure and standardized ways to:
- Generate and recover accounts using BIP39 mnemonic phrases
- Derive multiple accounts from a single seed using BIP32/BIP44
- Work with raw private keys for direct key management
- Generate P2TR (Taproot) addresses for modern Bitcoin transactions

## Key Concepts

### Mnemonic Phrases (BIP39)

Mnemonic phrases are human-readable representations of cryptographic seeds. They typically consist of 12, 15, 18, 21, or 24 words selected from a standardized wordlist. These phrases allow users to backup and recover their entire wallet with a simple list of words.

**Why use mnemonics?**
- Easy to write down and backup securely
- More user-friendly than raw hexadecimal seeds
- Standardized across different wallet implementations
- Support for multiple languages

### Hierarchical Deterministic Wallets (BIP32/BIP44)

HD wallets allow you to generate a tree of key pairs from a single seed. This means:
- One backup (mnemonic) protects unlimited addresses
- Deterministic address generation - same seed always produces same addresses
- Organized account structure following BIP44 derivation paths
- Enhanced privacy through address reuse prevention

**Standard derivation path:** `m/44'/0'/0'/0/0`
- `44'` - BIP44 specification
- `0'` - Bitcoin (coin type)
- `0'` - Account index
- `0` - External chain (receiving addresses)
- `0` - Address index

### Private Keys

Private keys are the fundamental cryptographic secret that controls Bitcoin funds. They are 256-bit numbers that must be kept secure. Anyone with access to a private key can spend the associated funds.

### P2TR (Pay-to-Taproot) Addresses

Taproot is Bitcoin's latest address format (activated November 2021) that provides:
- Enhanced privacy by making complex scripts look like simple transactions
- Lower transaction fees for multi-signature and smart contract scenarios
- Schnorr signatures for improved security and efficiency
- Addresses starting with "bc1p" on mainnet

## Implementation Guide

### Working with Mnemonics

The mnemonic module provides functions for generating and validating BIP39 mnemonic phrases:

```typescript
import { generateMnemonic, validateMnemonic, mnemonicToSeed } from './sdk/accounts/mnemonic';

// Generate a new 12-word mnemonic
const mnemonic = generateMnemonic(128); // 128 bits = 12 words
console.log('Mnemonic:', mnemonic);

// Validate the mnemonic
if (validateMnemonic(mnemonic)) {
  console.log('Valid mnemonic phrase');
}

// Convert mnemonic to seed for key derivation
const seed = await mnemonicToSeed(mnemonic, 'optional-passphrase');
```

**Entropy strength options:**
- 128 bits → 12 words
- 160 bits → 15 words
- 192 bits → 18 words
- 224 bits → 21 words
- 256 bits → 24 words (most secure)

### Using HD Keys

The HD key module implements BIP32 hierarchical deterministic key derivation:

```typescript
import { HDKey } from './sdk/accounts/hdKey';

// Create HD key from mnemonic
const mnemonic = 'your twelve word mnemonic phrase here...';
const hdKey = HDKey.fromMnemonic(mnemonic);

// Derive specific account using BIP44 path
const accountPath = "m/44'/0'/0'/0/0";
const account = hdKey.derive(accountPath);

// Get the private key
const privateKey = account.privateKey;

// Generate P2TR address
const address = account.getP2TRAddress('mainnet');
console.log('Taproot address:', address);

// Derive multiple addresses
for (let i = 0; i < 5; i++) {
  const childKey = hdKey.derive(`m/44'/0'/0'/0/${i}`);
  console.log(`Address ${i}:`, childKey.getP2TRAddress('mainnet'));
}
```

### Working with Private Keys

For scenarios where you need direct control over private keys:

```typescript
import { PrivateKey } from './sdk/accounts/privateKey';

// Generate a new random private key
const privateKey = PrivateKey.random();

// Create from existing private key hex
const existingKey = PrivateKey.fromHex('your-private-key-hex');

// Get public key
const publicKey = privateKey.getPublicKey();

// Generate P2TR address
const address = privateKey.getP2TRAddress('mainnet');

// Sign a message
const message = 'Hello, Bitcoin!';
const signature = privateKey.sign(message);

// Export private key
const privateKeyHex = privateKey.toHex();
const privateKeyWIF = privateKey.toWIF('mainnet'); // Wallet Import Format
```

## Security Best Practices

### Mnemonic Security

1. **Never store mnemonics in plain text** - Use encrypted storage or hardware wallets
2. **Write down physical backups** - Store in secure, separate locations
3. **Use strong passphrases** - Add BIP39 passphrase for additional security layer
4. **Verify backups** - Always test recovery before funding wallet

### Private Key Management

1. **Never expose private keys** - Don't log, transmit, or store in insecure locations
2. **Use environment variables** - For development, never hardcode keys
3. **Implement key rotation** - Regularly move funds to new addresses
4. **Use hardware wallets** - For production and high-value scenarios

### Address Generation

1. **Use new addresses** - Generate fresh addresses for each transaction
2. **Verify addresses** - Always double-check before sending funds
3. **Test with small amounts** - Verify address generation on testnet first
4. **Monitor for reuse** - Track and prevent address reuse

## Advanced Usage

### Multi-Account Wallet

```typescript
// Create multiple accounts from single mnemonic
const mnemonic = generateMnemonic(256);
const hdKey = HDKey.fromMnemonic(mnemonic);

// Generate accounts for different purposes
const accounts = {
  personal: hdKey.derive("m/44'/0'/0'/0/0"),
  savings: hdKey.derive("m/44'/0'/1'/0/0"),
  business: hdKey.derive("m/44'/0'/2'/0/0")
};

for (const [name, account] of Object.entries(accounts)) {
  console.log(`${name}:`, account.getP2TRAddress('mainnet'));
}
```

### Watch-Only Wallets

```typescript
// Create watch-only wallet using extended public key
const hdKey = HDKey.fromMnemonic(mnemonic);
const accountNode = hdKey.derive("m/44'/0'/0'");

// Export extended public key (safe to share for monitoring)
const xpub = accountNode.toExtendedPublicKey();

// Import xpub on monitoring system (cannot spend)
const watchOnly = HDKey.fromExtendedPublicKey(xpub);
const addresses = [];
for (let i = 0; i < 20; i++) {
  addresses.push(watchOnly.derive(`0/${i}`).getP2TRAddress('mainnet'));
}
```

## Network Support

The account management system supports multiple Bitcoin networks:

- **mainnet** - Bitcoin production network
- **testnet** - Bitcoin test network for development
- **regtest** - Local regression testing network

Always specify the correct network when generating addresses:

```typescript
const mainnetAddr = account.getP2TRAddress('mainnet');
const testnetAddr = account.getP2TRAddress('testnet');
```

## Related Documentation

For deeper understanding of account management:
- Review the implementation files listed in `relatedFiles`
- Study BIP39 specification for mnemonic generation
- Read BIP32 for HD key derivation details
- Understand BIP44 for account structure
- Learn about BIP341/BIP342 for Taproot implementation

## Common Pitfalls

1. **Network mismatch** - Using testnet keys on mainnet or vice versa
2. **Incorrect derivation paths** - Not following BIP44 standard
3. **Missing backups** - Generating keys without secure backup
4. **Passphrase confusion** - Forgetting BIP39 passphrase renders wallet unrecoverable
5. **Key exposure** - Accidentally logging or transmitting private keys

## Troubleshooting

**Invalid mnemonic:** Ensure you're using standard BIP39 wordlist and correct word count

**Wrong addresses generated:** Verify derivation path and network parameter

**Cannot recover wallet:** Check for BIP39 passphrase, verify word order and spelling

**Signature verification fails:** Ensure you're using the correct private key for the address
