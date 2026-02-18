---
title: RPC and HTTP Schema Definitions
slug: rpc-schemas
description: Type definitions and validation schemas for Kontor protocol communication
order: 12
category: architecture
tags: [typescript, rpc, http, schema, type-safety, validation]
relatedFiles: [src/sdk/types/rpc-schema.ts, src/sdk/types/http-schema.ts, src/wit/type-utils.ts]
---

# RPC and HTTP Schema Definitions

## Overview

Kontor provides strongly-typed schema definitions for both JSON-RPC and HTTP communication protocols. These schemas enable compile-time type checking, autocomplete support, and runtime validation for client-server communication. By defining the shape of requests and responses upfront, developers get immediate feedback about API usage and prevent common integration errors.

These schema systems serve as the foundation for type-safe communication in the Kontor SDK, ensuring that method calls, parameters, and return types are validated at both compile-time and runtime.

## RPC Schema System

### Core Types

The RPC schema system is built around the `RpcSchemaEntry` type, which defines the contract for a single RPC method:

```typescript
export type RpcSchemaEntry<
  M extends string = string,
  P = unknown,
  R = unknown,
> = {
  readonly Method: M;
  readonly Parameters?: P | undefined;
  readonly ReturnType: R;
};
```

Each entry specifies:
- **Method**: The RPC method name (e.g., `"user.create"`, `"wallet.getBalance"`)
- **Parameters**: The expected input parameters type
- **ReturnType**: The type of data returned by the method

### Schema Definition

An RPC schema is a readonly array of schema entries:

```typescript
export type RpcSchema = readonly RpcSchemaEntry[];

// Example usage:
const MyAppSchema = [
  {
    Method: "user.create",
    Parameters: { name: string; email: string },
    ReturnType: { id: string; name: string; email: string }
  },
  {
    Method: "user.get",
    Parameters: { id: string },
    ReturnType: { id: string; name: string; email: string } | null
  }
] as const;
```

### Type-Safe Request Function

The `RpcRequestFn` type provides a type-safe interface for making RPC calls:

```typescript
export type RpcRequestFn<Schema extends RpcSchema | undefined> = <
  S extends RpcSchema = Schema extends RpcSchema ? Schema : RpcSchema,
  M extends S[number]["Method"] = S[number]["Method"],
>(
  args: Schema extends RpcSchema
    ? {
        method: M;
        params: ExtractMethodEntry<S, M>["Parameters"];
      }
    : {
        method: string;
        params?: unknown;
      },
  options?: RpcRequestOptions | undefined,
) => Promise<
  Schema extends RpcSchema ? ExtractMethodEntry<S, M>["ReturnType"] : unknown
>;
```

When a schema is provided, TypeScript automatically:
- Restricts `method` to valid method names from the schema
- Enforces correct parameter types for the chosen method
- Types the return value according to the schema

### Request Options

The `RpcRequestOptions` type configures request behavior:

```typescript
export type RpcRequestOptions = {
  /** Deduplicate in-flight requests. */
  dedupe?: boolean | undefined;
  /** Methods to include or exclude from executing RPC requests. */
  methods?:
    | OneOf<
        | {
            include?: string[] | undefined;
          }
        | {
            exclude?: string[] | undefined;
          }
      >
    | undefined;
  /** The base delay (in ms) between retries. */
  retryDelay?: number | undefined;
  /** The max number of times to retry. */
  retryCount?: number | undefined;
  /** Unique identifier for the request. */
  uid?: string | undefined;
};
```

Key features:
- **Deduplication**: Prevents duplicate concurrent requests
- **Method filtering**: Include or exclude specific methods (mutually exclusive via `OneOf`)
- **Retry logic**: Configurable retry attempts and delays
- **Request tracking**: Unique identifiers for request management

### Method Extraction

The schema system provides utility types for extracting specific method information:

```typescript
export type ExtractMethodEntry<
  S extends RpcSchema,
  M extends S[number]["Method"],
> = Extract<S[number], { Method: M }>;

// Usage example:
type UserCreateEntry = ExtractMethodEntry<typeof MyAppSchema, "user.create">;
// Results in the specific schema entry for "user.create"
```

## HTTP Schema System

### HTTP Methods and Parameters

The HTTP schema supports standard HTTP methods:

```typescript
export type HttpMethod =
  | "GET"
  | "POST"
  | "PUT"
  | "PATCH"
  | "DELETE"
  | "HEAD"
  | "OPTIONS";
```

Parameters are organized into three categories:

```typescript
export type PathParams = Record<string, string | number>;
export type OrderedPathParams = readonly (readonly [string, string | number])[];
export type QueryParams = Record<
  string,
  string | number | boolean | (string | number | boolean)[] | undefined
>;
export type BodyParams = unknown;
```

The `HttpParameters` container bundles all parameter types:

```typescript
export type HttpParameters<
  Path extends PathParams | OrderedPathParams | undefined =
    | PathParams
    | OrderedPathParams
    | undefined,
  Query extends QueryParams | undefined = QueryParams | undefined,
  Body extends BodyParams | undefined = BodyParams | undefined,
> = {
  path: Path;
  query: Query;
  body: Body;
};
```

### HTTP Schema Entry

Each HTTP endpoint is defined using `HttpSchemaEntry`:

```typescript
export type HttpSchemaEntry<
  R extends string = string,
  M extends HttpMethod = HttpMethod,
  P extends HttpParameters = HttpParameters,
  T = unknown,
> = {
  readonly Route: R;
  readonly Method: M;
  readonly Parameters?: P | undefined;
  readonly ReturnType: T;
};
```

Example schema definition:

```typescript
const ApiSchema = [
  {
    Route: "/users/:userId/posts/:postId",
    Method: "GET",
    Parameters: {
      path: { userId: string, postId: string },
      query: { include?: string[] },
      body: undefined
    },
    ReturnType: { id: string; title: string; content: string }
  },
  {
    Route: "/users/:userId",
    Method: "PATCH",
    Parameters: {
      path: { userId: string },
      query: undefined,
      body: { name?: string; email?: string }
    },
    ReturnType: { id: string; name: string; email: string }
  }
] as const;
```

### Parameter Extraction

The HTTP schema provides sophisticated type extraction utilities:

```typescript
export type ExtractRouteEntry<
  S extends HttpSchema,
  R extends S[number]["Route"],
  M extends S[number]["Method"] | undefined = undefined,
> = [M] extends [undefined]
  ? Extract<S[number], { Route: R }>
  : Extract<S[number], { Route: R; Method: M }>;

export type ExtractMethod<
  S extends HttpSchema,
  R extends S[number]["Route"],
> = ExtractRouteEntry<S, R>["Method"];

export type ExtractPathParams<
  S extends HttpSchema,
  R extends S[number]["Route"],
> =
  ExtractRouteEntry<S, R>["Parameters"] extends HttpParameters<
    infer Path,
    any,
    any
  >
    ? Path
    : never;

export type ExtractQueryParams<
  S extends HttpSchema,
  R extends S[number]["Route"],
> =
  ExtractRouteEntry<S, R>["Parameters"] extends HttpParameters<
    any,
    infer Query,
    any
  >
    ? Query
    : never;

export type ExtractBodyParams<
  S extends HttpSchema,
  R extends S[number]["Route"],
> =
  ExtractRouteEntry<S, R>["Parameters"] extends HttpParameters<
    any,
    any,
    infer Body
  >
    ? Body
    : never;
```

### Path Parameter Ordering

The schema supports both record-based and ordered path parameters:

```typescript
// Record-based (unordered)
type UserParams = { userId: string; postId: string };

// Ordered (preserves parameter order in URL)
type OrderedUserParams = readonly [
  readonly ["userId", string],
  readonly ["postId", string]
];
```

The `OrderedToRecord` helper converts ordered params to record form:

```typescript
type OrderedToRecord<T extends OrderedPathParams> = {
  [K in T[number][0]]: Extract<T[number], readonly [K, any]>[1];
};
```

### Required vs Optional Parameters

The schema system distinguishes between required and optional parameters:

```typescript
type IsRequired<T> = undefined extends T ? false : true;

type NormalizePathParams<P> = P extends OrderedPathParams
  ? OrderedToRecord<P>
  : P extends PathParams
    ? P
    : never;
```

This enables TypeScript to enforce that required parameters must be provided while optional ones can be omitted.

## Key Advantages

### Type Safety

Both schema systems provide compile-time guarantees:
- Method/route names are validated against the schema
- Parameter types are enforced
- Return types are automatically inferred
- Invalid combinations are caught at compile time

### Developer Experience

- **Autocomplete**: IDEs suggest available methods, routes, and parameters
- **Inline documentation**: Types serve as living documentation
- **Refactoring safety**: Changes to schemas propagate through the codebase
- **Error prevention**: Typos and type mismatches are caught immediately

### Runtime Flexibility

While providing strong typing, the schemas support:
- Optional schema usage (falls back to `unknown` types)
- Dynamic method filtering
- Request deduplication and retry logic
- Custom request identifiers

## Integration Points

These schema definitions integrate with:
- **Client SDK**: Type-safe request builders
- **Server handlers**: Route and method validation
- **Middleware**: Request/response transformation
- **Testing utilities**: Mock generation and validation

## Best Practices

1. **Define schemas as const**: Use `as const` to preserve literal types
2. **Keep schemas co-located**: Place schemas near the code that uses them
3. **Version schemas**: Track breaking changes to API contracts
4. **Document parameter constraints**: Use JSDoc comments for additional context
5. **Use ordered params when order matters**: Especially for REST-style URLs
6. **Leverage type extraction**: Use helper types to derive related types

These schema systems form the backbone of Kontor's type-safe communication layer, enabling developers to build robust, maintainable applications with confidence.
