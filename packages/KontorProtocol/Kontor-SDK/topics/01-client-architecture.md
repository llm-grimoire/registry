---
title: Client Architecture Overview
slug: client-architecture
description: Understanding the client-server model, transport layers, and how public and wallet clients work together
order: 1
category: architecture
tags: [clients, architecture, transports, http, rpc]
relatedFiles: [src/sdk/clients/transports/http.ts, src/sdk/clients/transports/create-http-transport.js, src/sdk/utils/http.js, src/sdk/types/http-schema.ts, src/sdk/errors/transport.js]
---

# Client Architecture Overview

## Overview

The SDK implements a modular client-server architecture that separates concerns between communication layers (transports) and interaction patterns (clients). This design allows you to:

- **Swap transport mechanisms** without changing client code
- **Configure network behavior** like retries, timeouts, and fetch options
- **Type-safe API interactions** through HTTP schemas
- **Handle multiple chains** with different RPC endpoints

The architecture follows a layered approach where clients depend on transports, and transports handle the low-level communication details.

## Core Architectural Layers

### 1. Transport Layer

Transports are responsible for the actual network communication. They abstract away the details of HTTP requests, WebSocket connections, or other communication protocols.

#### HTTP Transport

The HTTP transport is the most common transport layer, handling REST API and JSON-RPC communication:

```typescript
import { http } from './sdk/clients/transports/http';

// Create an HTTP transport with default settings
const transport = http('https://api.example.com');

// Create with custom configuration
const customTransport = http('https://api.example.com', {
  timeout: 30_000,
  retryCount: 3,
  retryDelay: 1000,
  fetchOptions: {
    headers: {
      'Authorization': 'Bearer token'
    }
  }
});
```

#### Transport Configuration Options

The `HttpTransportConfig` type defines all available configuration options:

```typescript
type HttpTransportConfig = {
  // Custom fetch implementation (useful for testing or Node.js environments)
  fetchFn?: typeof fetch;
  
  // Standard fetch options (headers, credentials, etc.)
  fetchOptions?: RequestInit;
  
  // Lifecycle hooks for request/response interception
  onFetchRequest?: (request: Request) => void;
  onFetchResponse?: (response: Response) => void;
  
  // Transport identification
  key?: string;  // Default: 'http'
  name?: string; // Default: 'HTTP Transport'
  
  // Retry configuration
  retryCount?: number;    // Default: from client
  retryDelay?: number;    // Base delay in ms between retries
  
  // Type safety for API endpoints
  httpSchema?: HttpSchema;
  
  // Request timeout in milliseconds
  timeout?: number; // Default: 10_000
};
```

### 2. Transport Creation and Initialization

Transports are factory functions that return configured transport instances. The HTTP transport implementation shows this pattern:

```typescript
// From src/sdk/clients/transports/http.ts
export function http<httpSchema extends HttpSchema | undefined = undefined>(
  url?: string | undefined,
  config: HttpTransportConfig<httpSchema> = {},
): HttpTransport<httpSchema> {
  const {
    fetchFn,
    fetchOptions,
    key = "http",
    name = "HTTP Transport",
    onFetchRequest,
    onFetchResponse,
    retryDelay,
  } = config;

  return Object.assign(
    ({
      chain,
      retryCount: retryCount_,
      timeout: timeout_,
    }: Parameters<HttpTransport<httpSchema>>[0]) => {
      const retryCount = config.retryCount ?? retryCount_;
      const timeout = timeout_ ?? config.timeout ?? 10_000;
      const url_ = url || chain?.urls.default.http[0];
      if (!url_) throw new UrlRequiredError();

      const httpClient = getHttpClient(url_, {
        fetchFn,
        fetchOptions,
        onRequest: onFetchRequest,
        onResponse: onFetchResponse,
        timeout,
      });

      return createHttpTransport(
        {
          key,
          name,
          async request({ route, method, path, query, body }) {
            return httpClient.request({ route, method, body, path, query });
          },
          retryCount,
          retryDelay,
          timeout,
          type: "http" as const,
        },
        {
          fetchOptions,
          url: url_,
        },
      );
    },
    {
      [TRANSPORT_TYPE]: "http",
    },
  );
}
```

### 3. URL Resolution Strategy

The transport layer implements a flexible URL resolution strategy:

1. **Explicit URL**: If provided to `http()`, use it directly
2. **Chain Default**: Fall back to `chain?.urls.default.http[0]`
3. **Error**: Throw `UrlRequiredError` if neither is available

```typescript
const url_ = url || chain?.urls.default.http[0];
if (!url_) throw new UrlRequiredError();
```

This allows clients to work with chain configurations without hardcoding URLs.

### 4. HTTP Client Abstraction

The transport delegates to an HTTP client (`getHttpClient`) that handles:

- **Request formatting**: Converting high-level requests to fetch calls
- **Response handling**: Parsing and error handling
- **Timeouts**: Aborting requests that exceed time limits
- **Lifecycle hooks**: Calling `onRequest` and `onResponse` callbacks

```typescript
const httpClient = getHttpClient(url_, {
  fetchFn,
  fetchOptions,
  onRequest: onFetchRequest,
  onResponse: onFetchResponse,
  timeout,
});
```

### 5. Request Interface

Transports expose a unified request interface that abstracts HTTP details:

```typescript
async request({ route, method, path, query, body }) {
  return httpClient.request({ 
    route,   // API route/endpoint
    method,  // HTTP method (GET, POST, etc.)
    body,    // Request body
    path,    // Path parameters
    query    // Query string parameters
  });
}
```

This interface allows clients to make requests without knowing transport implementation details.

## Client Types

While the transport layer handles communication, clients provide high-level APIs for specific use cases:

### Public Clients

Public clients interact with read-only blockchain data and public APIs. They use transports to:
- Query blockchain state
- Read smart contract data
- Fetch transaction history
- Monitor events

### Wallet Clients

Wallet clients handle operations requiring authentication or signing:
- Sending transactions
- Signing messages
- Managing accounts
- Interacting with smart contracts that modify state

## Type Safety with HTTP Schemas

The architecture supports typed API interactions through `HttpSchema`:

```typescript
type MyAPISchema = {
  '/users/:id': {
    GET: {
      params: { id: string };
      response: { name: string; email: string };
    };
  };
};

const transport = http<MyAPISchema>('https://api.example.com', {
  httpSchema: {} as MyAPISchema
});
```

This provides compile-time type checking for request parameters and response types.

## Retry and Error Handling

The architecture implements configurable retry logic:

```typescript
const transport = http('https://api.example.com', {
  retryCount: 3,      // Retry up to 3 times
  retryDelay: 1000,   // Wait 1 second between retries
  timeout: 10_000,    // 10 second timeout per request
});
```

Retries are handled at the transport layer, transparently to the client code.

## Custom Fetch Implementation

For Node.js environments or testing, you can provide a custom fetch implementation:

```typescript
import nodeFetch from 'node-fetch';

const transport = http('https://api.example.com', {
  fetchFn: nodeFetch as any,
});
```

## Lifecycle Hooks

Intercept requests and responses for logging, metrics, or modification:

```typescript
const transport = http('https://api.example.com', {
  onFetchRequest: (request) => {
    console.log('Request:', request.url);
  },
  onFetchResponse: (response) => {
    console.log('Response:', response.status);
  },
});
```

## Best Practices

1. **Reuse transports**: Create one transport instance per endpoint and reuse it
2. **Configure timeouts**: Set appropriate timeouts based on your API's characteristics
3. **Use retry logic**: Configure retries for transient failures
4. **Type your schemas**: Define `HttpSchema` types for type-safe API interactions
5. **Handle errors**: Always handle `UrlRequiredError` and transport-level errors
6. **Use lifecycle hooks**: Implement logging and monitoring through hooks

## Error Types

The transport layer defines specific error types:

```typescript
type HttpTransportErrorType =
  | CreateTransportErrorType    // Transport creation errors
  | UrlRequiredErrorType        // Missing URL configuration
  | ErrorType;                  // General errors
```

Always handle these errors appropriately in client code:

```typescript
try {
  const transport = http(undefined); // Will throw UrlRequiredError
} catch (error) {
  if (error instanceof UrlRequiredError) {
    // Handle missing URL
  }
}
```

## Summary

The client architecture provides:
- **Modularity**: Swap transports without changing client code
- **Flexibility**: Configure timeouts, retries, and custom fetch implementations
- **Type Safety**: Schema-based type checking for API interactions
- **Extensibility**: Lifecycle hooks for monitoring and modification
- **Error Handling**: Well-defined error types and retry logic

This design enables building robust, maintainable applications that can adapt to different network conditions and API requirements.
