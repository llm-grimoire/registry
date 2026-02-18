---
title: Transport Layer Abstraction
slug: transport-layers
description: HTTP, RPC, and custom transport implementations for blockchain communication
order: 3
category: architecture
tags: [transport, http, rpc, client, architecture, abstraction]
relatedFiles: [src/sdk/clients/transports/http.ts, src/sdk/clients/transports/create-http-transport.js, src/sdk/utils/http.js, src/sdk/types/http-schema.js]
---

# Transport Layer Abstraction

## Overview

The transport layer provides an abstraction for communication between blockchain clients and external services. It decouples the SDK's core client logic from the underlying communication protocol, enabling flexibility in how requests are made and responses are handled.

The transport layer supports multiple protocols including HTTP, RPC, and custom implementations. This architecture allows you to:

- Switch between different communication protocols without changing client code
- Configure request/response handling with callbacks and middleware
- Implement custom retry logic and timeout management
- Type-safe request schemas for API endpoints
- Chain-specific transport configurations

## Key Concepts

### Transport Types

Transports are identified by a `TRANSPORT_TYPE` symbol that distinguishes between different implementations:

- **HTTP Transport**: RESTful API communication over HTTP/HTTPS
- **RPC Transport**: JSON-RPC protocol support for blockchain nodes
- **Custom Transports**: User-defined implementations for specialized protocols

### Transport Configuration

Each transport accepts a configuration object that controls its behavior:

```typescript
type HttpTransportConfig<httpSchema extends HttpSchema | undefined = undefined> = {
  fetchFn?: HttpClientOptions["fetchFn"] | undefined;
  fetchOptions?: HttpClientOptions["fetchOptions"] | undefined;
  onFetchRequest?: HttpClientOptions["onRequest"] | undefined;
  onFetchResponse?: HttpClientOptions["onResponse"] | undefined;
  key?: string | undefined;
  name?: string | undefined;
  retryCount?: number | undefined;
  retryDelay?: number | undefined;
  httpSchema?: httpSchema | HttpSchema | undefined;
  timeout?: number | undefined;
};
```

### Transport Factory Pattern

Transports use a factory pattern that returns a function. This allows for lazy initialization and runtime configuration:

```typescript
// Factory function that creates the transport
const transport = http("https://api.example.com", {
  timeout: 5000,
  retryCount: 3
});

// Transport is initialized when called with chain and runtime config
const initialized = transport({
  chain: myChain,
  retryCount: 2,
  timeout: 10000
});
```

## HTTP Transport Implementation

The HTTP transport is the primary implementation for RESTful API communication.

### Basic Usage

```typescript
import { http } from "./transports/http.js";

// Create HTTP transport with URL
const transport = http("https://api.blockchain.com");

// With configuration
const configuredTransport = http("https://api.blockchain.com", {
  timeout: 10000,
  retryCount: 3,
  retryDelay: 1000,
  key: "my-http-transport",
  name: "Custom HTTP Transport"
});
```

### Request/Response Callbacks

The transport supports lifecycle hooks for request and response handling:

```typescript
const transport = http("https://api.blockchain.com", {
  onFetchRequest: (request) => {
    console.log("Making request:", request);
    // Modify request headers, log, etc.
  },
  onFetchResponse: (response) => {
    console.log("Received response:", response);
    // Log response, handle errors, etc.
  }
});
```

### Custom Fetch Implementation

You can provide a custom fetch function for specialized environments:

```typescript
const transport = http("https://api.blockchain.com", {
  fetchFn: async (url, options) => {
    // Custom fetch implementation
    // Useful for environments without native fetch
    return customFetch(url, options);
  }
});
```

### Typed HTTP Schema

Define type-safe request schemas for your API endpoints:

```typescript
type MyAPISchema = {
  "/blocks": {
    method: "GET";
    query: { limit: number; offset: number };
    response: { blocks: Block[] };
  };
  "/transactions": {
    method: "POST";
    body: { from: string; to: string; amount: bigint };
    response: { txHash: string };
  };
};

const transport = http<MyAPISchema>("https://api.blockchain.com", {
  httpSchema: {} as MyAPISchema
});
```

## Transport Initialization

The HTTP transport factory creates a transport instance through a two-step process:

### Step 1: Factory Configuration

```typescript
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
    ({ chain, retryCount: retryCount_, timeout: timeout_ }) => {
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

      return createHttpTransport(...);
    },
    {
      [TRANSPORT_TYPE]: "http",
    },
  );
}
```

### Step 2: Runtime Initialization

When the transport function is called, it:

1. **Resolves Configuration**: Merges factory config with runtime parameters
2. **Determines URL**: Uses provided URL or falls back to chain default
3. **Creates HTTP Client**: Initializes the underlying HTTP client with all options
4. **Returns Transport**: Creates the actual transport with request handling

```typescript
const initialized = transport({
  chain: ethereumChain,
  retryCount: 2,
  timeout: 5000
});
```

## Error Handling

The transport layer includes comprehensive error handling:

### URL Validation

```typescript
const url_ = url || chain?.urls.default.http[0];
if (!url_) throw new UrlRequiredError();
```

Ensures that a valid URL is always provided, either explicitly or through chain configuration.

### Transport Error Types

```typescript
export type HttpTransportErrorType =
  | CreateTransportErrorType
  | UrlRequiredErrorType
  | ErrorType;
```

All transport errors are typed, making error handling predictable and type-safe.

## Retry and Timeout Configuration

### Retry Logic

Configure automatic retry behavior for failed requests:

```typescript
const transport = http("https://api.blockchain.com", {
  retryCount: 3,        // Retry up to 3 times
  retryDelay: 1000     // Wait 1 second between retries
});
```

### Timeout Management

```typescript
const transport = http("https://api.blockchain.com", {
  timeout: 10000  // 10 second timeout (default)
});

// Override at initialization
const initialized = transport({
  chain: myChain,
  timeout: 5000  // 5 second timeout
});
```

## Advanced Patterns

### Chain-Specific Transports

Let the chain configuration determine the transport URL:

```typescript
const transport = http();  // No URL provided

const initialized = transport({
  chain: {
    urls: {
      default: {
        http: ["https://mainnet.blockchain.com"]
      }
    }
  }
});
```

### Custom Fetch Options

Pass additional options to the underlying fetch call:

```typescript
const transport = http("https://api.blockchain.com", {
  fetchOptions: {
    headers: {
      "Authorization": "Bearer token",
      "X-API-Key": "my-api-key"
    },
    mode: "cors",
    credentials: "include"
  }
});
```

### Request Method Handling

The transport supports various HTTP methods and request types:

```typescript
return createHttpTransport(
  {
    key,
    name,
    async request({ route, method, path, query, body }) {
      return httpClient.request({ 
        route,   // Full route path
        method,  // GET, POST, PUT, DELETE, etc.
        body,    // Request body for POST/PUT
        path,    // Path parameters
        query    // Query string parameters
      });
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
```

## Best Practices

1. **Always Configure Timeouts**: Set reasonable timeout values to prevent hanging requests
2. **Use Typed Schemas**: Define HTTP schemas for type safety and better developer experience
3. **Implement Callbacks**: Use `onFetchRequest` and `onFetchResponse` for logging and monitoring
4. **Chain Configuration**: Prefer chain-based URLs for multi-network support
5. **Error Handling**: Always handle `UrlRequiredError` when URLs might be missing
6. **Retry Strategy**: Configure retry logic based on your API's reliability and rate limits

## Integration with Clients

Transports integrate seamlessly with SDK clients:

```typescript
import { createClient } from "./client.js";
import { http } from "./transports/http.js";

const client = createClient({
  chain: ethereumChain,
  transport: http("https://eth-mainnet.example.com", {
    timeout: 10000,
    retryCount: 3
  })
});
```

The transport layer handles all communication details, allowing clients to focus on business logic while maintaining flexibility in how network requests are executed.
