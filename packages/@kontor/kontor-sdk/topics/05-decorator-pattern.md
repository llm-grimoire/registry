---
title: Client Extension with Decorators
slug: decorator-pattern
description: Extending client functionality using the decorator pattern for modular action composition
order: 5
category: patterns
tags: [decorators, clients, design-patterns, actions, extensibility]
relatedFiles: [src/sdk/clients/decorators/public.ts, src/sdk/clients/decorators/wallet.ts, src/sdk/clients/create-public-client.js, src/sdk/clients/create-wallet-client.js]
---

## Overview

The decorator pattern is a structural design pattern that allows you to dynamically add functionality to client instances without modifying their core implementation. In this SDK, decorators provide a clean and composable way to extend client capabilities by attaching action methods.

This pattern enables:
- **Separation of concerns**: Actions are defined independently from client implementations
- **Modularity**: Add only the functionality you need to each client
- **Type safety**: Full TypeScript support with generic constraints
- **Composability**: Chain multiple decorators together

## Key Concepts

### Decorator Functions

Decorators are higher-order functions that take a client instance and return an object containing action methods. Each action method internally calls the corresponding action function with the client as the first parameter.

### Client Types

The SDK provides different decorator sets for different client types:

- **PublicActions**: Read-only blockchain operations (transactions, mempool)
- **WalletActions**: Wallet-specific operations (signing, address management)

## Public Actions Decorator

The `publicActions` decorator extends clients with public blockchain operations:

```typescript
import type { Account } from "../../accounts/types.js";
import type { Chain } from "../../types/chain.js";
import type { Address, XOnlyPubKey } from "../../types/misc.js";
import type { RpcTransport } from "../transports/create-rpc-transport.js";
import type { PublicClient as PublicClient } from "../create-public-client.js";

import {
  type SendRawTransactionParams,
  type SendRawTransactionReturnType,
  sendRawTransaction,
} from "../../actions/public/send-raw-transaction.js";

import {
  type TestMempoolAcceptParams,
  type TestMempoolAcceptReturnType,
  testMempoolAccept,
} from "../../actions/public/test-mempool-accept.js";

export type PublicActions<
  _chain extends Chain | undefined = Chain | undefined,
  _account extends Account | [Address, XOnlyPubKey] | undefined =
    | Account
    | [Address, XOnlyPubKey]
    | undefined,
> = {
  sendRawTransaction: (
    parameters: SendRawTransactionParams,
  ) => Promise<SendRawTransactionReturnType>;
  testMempoolAccept: (
    parameters: TestMempoolAcceptParams,
  ) => Promise<TestMempoolAcceptReturnType>;
};

export function publicActions<
  transport extends RpcTransport = RpcTransport,
  chain extends Chain | undefined = Chain | undefined,
  account extends Account | undefined = Account | undefined,
>(
  client: PublicClient<transport, chain, account>,
): PublicActions<chain, account> {
  return {
    sendRawTransaction: (args) => sendRawTransaction(client, args),
    testMempoolAccept: (args) => testMempoolAccept(client, args),
  };
}
```

### Available Public Actions

- **sendRawTransaction**: Broadcasts a raw transaction to the network
- **testMempoolAccept**: Tests whether transactions would be accepted by the mempool

## Wallet Actions Decorator

The `walletActions` decorator extends clients with wallet-specific operations:

```typescript
import type { Account } from "../../accounts/types.js";
import type { Chain } from "../../types/chain.js";
import type { RpcTransport } from "../transports/create-rpc-transport.js";
import type { WalletClient } from "../create-wallet-client.js";

import {
  getAddresses,
  type GetAddressesReturnType,
} from "../../actions/wallet/get-addresses.js";

import {
  signPsbt,
  type SignPsbtReturnType,
  type SignPsbtParameters,
} from "../../actions/wallet/sign-psbt.js";

export type WalletActions<
  chain extends Chain | undefined = Chain | undefined,
  account extends Account | undefined = Account | undefined,
> = {
  getAddresses: () => Promise<GetAddressesReturnType>;
  signPsbt: (
    params: SignPsbtParameters<chain, account>,
  ) => Promise<SignPsbtReturnType>;
};

export function walletActions<
  transport extends RpcTransport,
  chain extends Chain | undefined = Chain | undefined,
  account extends Account | undefined = Account | undefined,
>(
  client: WalletClient<transport, chain, account>,
): WalletActions<chain, account> {
  return {
    getAddresses: () => getAddresses(client),
    signPsbt: (params) => signPsbt(client, params),
  };
}
```

### Available Wallet Actions

- **getAddresses**: Retrieves addresses from the wallet
- **signPsbt**: Signs a Partially Signed Bitcoin Transaction (PSBT)

## Usage Patterns

### Basic Usage

Apply decorators to existing client instances:

```typescript
import { createPublicClient } from './clients/create-public-client.js';
import { publicActions } from './clients/decorators/public.js';

const client = createPublicClient({
  transport: http('https://rpc-url.com')
});

const extendedClient = publicActions(client);

// Now use the decorated methods
const txId = await extendedClient.sendRawTransaction({
  serializedTransaction: '0x...'
});
```

### Combining Decorators

You can apply multiple decorators to create clients with mixed capabilities:

```typescript
import { createWalletClient } from './clients/create-wallet-client.js';
import { walletActions } from './clients/decorators/wallet.js';
import { publicActions } from './clients/decorators/public.js';

const baseClient = createWalletClient({
  transport: http('https://rpc-url.com')
});

const fullClient = {
  ...baseClient,
  ...walletActions(baseClient),
  ...publicActions(baseClient)
};

// Access both wallet and public actions
const addresses = await fullClient.getAddresses();
const txId = await fullClient.sendRawTransaction({ /* ... */ });
```

## Type Safety

Decorators preserve full type information through generics:

- **transport**: The RPC transport type
- **chain**: The blockchain configuration (can be undefined)
- **account**: The account type (can be undefined)

This ensures that action methods receive proper type checking based on the client configuration:

```typescript
// Type parameters flow through the decorator
type MyActions = PublicActions<MyChain, MyAccount>;

// TypeScript knows the exact parameter and return types
const result: SendRawTransactionReturnType = 
  await client.sendRawTransaction(params);
```

## Benefits of the Decorator Pattern

1. **Flexibility**: Add only the methods you need to each client instance
2. **Testability**: Mock individual actions without affecting the client
3. **Maintainability**: Actions are defined separately and can evolve independently
4. **Tree-shaking**: Unused actions can be eliminated by bundlers
5. **Extensibility**: Create custom decorators for project-specific needs

## Creating Custom Decorators

You can create your own decorators following the same pattern:

```typescript
import type { PublicClient } from '../create-public-client.js';
import { myCustomAction } from '../../actions/custom/my-action.js';

export type CustomActions = {
  myCustomAction: (params: MyParams) => Promise<MyReturnType>;
};

export function customActions<
  transport extends RpcTransport,
  chain extends Chain | undefined = Chain | undefined,
  account extends Account | undefined = Account | undefined,
>(
  client: PublicClient<transport, chain, account>,
): CustomActions {
  return {
    myCustomAction: (params) => myCustomAction(client, params),
  };
}
```
