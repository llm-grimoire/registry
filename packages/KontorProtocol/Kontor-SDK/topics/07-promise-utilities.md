---
title: Advanced Promise Handling
slug: promise-utilities
description: Request batching, deduplication, and retry mechanisms for efficient RPC calls
order: 7
category: patterns
tags: [promises, performance, rpc, utilities, batching, deduplication, retry]
relatedFiles: [src/sdk/utils/promise/withBatch.ts, src/sdk/utils/promise/withDedupe.ts, src/sdk/utils/promise/withRetry.ts]
---

# Advanced Promise Handling

## Overview

The Stacks.js SDK provides advanced promise utilities designed to optimize RPC calls and network requests. These utilities address common challenges in blockchain applications:

- **Request Batching**: Combine multiple requests into a single network call
- **Deduplication**: Prevent redundant requests for the same data
- **Retry Logic**: Handle transient failures with exponential backoff

These patterns are essential for building efficient, responsive blockchain applications that minimize network overhead and improve user experience.

## Why Use Promise Utilities?

Blockchain RPC calls are often:
- **Expensive**: Network latency and rate limits make efficiency critical
- **Redundant**: Multiple components may request the same data simultaneously
- **Unreliable**: Network issues require robust retry mechanisms

The promise utilities solve these problems by wrapping standard promises with intelligent caching, batching, and retry logic.

## Key Concepts

### Request Batching (`withBatch`)

Batching combines multiple individual requests into a single batch request, reducing network overhead. This is particularly useful when:
- Loading data for multiple addresses or transactions
- Fetching account balances across multiple wallets
- Querying contract state for multiple contracts

**Benefits**:
- Reduced network calls (N requests → 1 batch request)
- Lower latency for bulk operations
- Better rate limit compliance
- Improved application responsiveness

**Use Cases**:
```typescript
// Instead of making 10 separate RPC calls
const balances = await Promise.all(
  addresses.map(addr => getBalance(addr))
);

// Batch them into a single request
const balances = await withBatch(
  addresses.map(addr => () => getBalance(addr))
);
```

### Request Deduplication (`withDedupe`)

Deduplication ensures that identical concurrent requests share a single underlying promise, preventing redundant network calls.

**Benefits**:
- Eliminates duplicate requests
- Reduces server load
- Guarantees consistent results
- Improves cache hit rates

**Use Cases**:
```typescript
// Multiple components requesting the same data
const Component1 = () => {
  const data = useQuery(() => withDedupe('user-123', fetchUser));
};

const Component2 = () => {
  const data = useQuery(() => withDedupe('user-123', fetchUser));
};

// Only one actual network request is made
```

### Retry Logic (`withRetry`)

Retry logic automatically retries failed requests with exponential backoff, handling transient network failures gracefully.

**Benefits**:
- Handles temporary network issues
- Exponential backoff prevents server overload
- Configurable retry strategies
- Improved reliability

**Use Cases**:
```typescript
// Retry failed RPC calls automatically
const transaction = await withRetry(
  () => broadcastTransaction(tx),
  { maxRetries: 3, backoff: 'exponential' }
);
```

## Implementation Patterns

### Combining Utilities

These utilities can be composed for powerful request optimization:

```typescript
import { withBatch, withDedupe, withRetry } from '@stacks/transactions';

// Combine deduplication with retry
const fetchWithDedupeAndRetry = (key: string, fn: () => Promise<any>) => {
  return withDedupe(key, () => withRetry(fn));
};

// Batch multiple deduplicated requests
const results = await withBatch(
  ids.map(id => () => 
    withDedupe(`resource-${id}`, () => fetchResource(id))
  )
);
```

### Batching Strategy

When implementing batching:

1. **Group related requests**: Batch requests that can be fulfilled by the same endpoint
2. **Set appropriate batch sizes**: Balance between latency and throughput
3. **Use time-based flushing**: Flush batches after a timeout to prevent indefinite waiting
4. **Handle partial failures**: Implement graceful degradation when some batch items fail

```typescript
// Conceptual batching implementation
class RequestBatcher {
  private queue: Array<() => Promise<any>> = [];
  private timeout: NodeJS.Timeout | null = null;
  
  async add<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          resolve(await fn());
        } catch (error) {
          reject(error);
        }
      });
      
      this.scheduleFlush();
    });
  }
  
  private scheduleFlush() {
    if (this.timeout) clearTimeout(this.timeout);
    
    this.timeout = setTimeout(() => this.flush(), 50);
    
    if (this.queue.length >= 10) {
      this.flush();
    }
  }
  
  private async flush() {
    const batch = this.queue.splice(0);
    if (this.timeout) clearTimeout(this.timeout);
    
    await Promise.all(batch.map(fn => fn()));
  }
}
```

### Deduplication Strategy

Effective deduplication requires:

1. **Stable cache keys**: Generate consistent keys for identical requests
2. **Appropriate TTL**: Clear cached promises after reasonable timeouts
3. **Error handling**: Decide whether to cache failures or retry
4. **Memory management**: Implement cache eviction for long-running applications

```typescript
// Conceptual deduplication implementation
class PromiseCache {
  private cache = new Map<string, Promise<any>>();
  
  async dedupe<T>(key: string, fn: () => Promise<T>): Promise<T> {
    if (this.cache.has(key)) {
      return this.cache.get(key)!;
    }
    
    const promise = fn()
      .finally(() => {
        // Clear from cache after resolution
        setTimeout(() => this.cache.delete(key), 5000);
      });
    
    this.cache.set(key, promise);
    return promise;
  }
}
```

### Retry Strategy

Robust retry logic includes:

1. **Exponential backoff**: Increase delay between retries (100ms, 200ms, 400ms...)
2. **Maximum retries**: Prevent infinite retry loops
3. **Retry conditions**: Only retry on transient errors (network, timeout, 5xx)
4. **Jitter**: Add randomness to prevent thundering herd

```typescript
// Conceptual retry implementation
interface RetryOptions {
  maxRetries?: number;
  initialDelay?: number;
  maxDelay?: number;
  backoffFactor?: number;
  retryCondition?: (error: Error) => boolean;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const {
    maxRetries = 3,
    initialDelay = 100,
    maxDelay = 5000,
    backoffFactor = 2,
    retryCondition = () => true
  } = options;
  
  let lastError: Error;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      if (attempt === maxRetries || !retryCondition(lastError)) {
        throw lastError;
      }
      
      const delay = Math.min(
        initialDelay * Math.pow(backoffFactor, attempt),
        maxDelay
      );
      
      // Add jitter (±25%)
      const jitter = delay * (0.75 + Math.random() * 0.5);
      
      await new Promise(resolve => setTimeout(resolve, jitter));
    }
  }
  
  throw lastError!;
}
```

## Best Practices

### 1. Choose the Right Utility

- **Use batching** when making multiple similar requests
- **Use deduplication** when the same data might be requested concurrently
- **Use retry** for critical operations that must succeed

### 2. Configure Appropriately

```typescript
// Conservative retry for user-facing operations
await withRetry(fetchData, {
  maxRetries: 2,
  initialDelay: 200,
  retryCondition: (error) => error.message.includes('timeout')
});

// Aggressive retry for background operations
await withRetry(syncData, {
  maxRetries: 5,
  initialDelay: 1000,
  maxDelay: 30000
});
```

### 3. Monitor Performance

Track metrics to validate optimization:
- Request count reduction (batching)
- Cache hit rate (deduplication)
- Retry success rate
- Average request latency

### 4. Handle Edge Cases

```typescript
// Handle batch partial failures
try {
  const results = await withBatch(requests);
} catch (error) {
  // Some requests may have succeeded
  console.error('Batch failed:', error);
  // Fall back to individual requests
}

// Clear deduplication cache on authentication changes
onAuthChange(() => {
  clearDedupeCache();
});

// Don't retry non-retryable errors
await withRetry(sendTransaction, {
  retryCondition: (error) => {
    // Don't retry if transaction is invalid
    return !error.message.includes('invalid transaction');
  }
});
```

## Integration Examples

### With React Query

```typescript
import { useQuery } from '@tanstack/react-query';
import { withDedupe, withRetry } from '@stacks/transactions';

function useStacksAccount(address: string) {
  return useQuery({
    queryKey: ['account', address],
    queryFn: () => withDedupe(
      `account-${address}`,
      () => withRetry(() => fetchAccount(address))
    ),
    staleTime: 30000,
  });
}
```

### With SWR

```typescript
import useSWR from 'swr';
import { withDedupe, withRetry } from '@stacks/transactions';

const fetcher = (url: string) => 
  withDedupe(url, () => 
    withRetry(() => fetch(url).then(r => r.json()))
  );

function useStacksData(endpoint: string) {
  return useSWR(`/api/stacks${endpoint}`, fetcher);
}
```

### Batch Loading Multiple Resources

```typescript
import { withBatch } from '@stacks/transactions';

async function loadDashboard(addresses: string[]) {
  // Batch all account fetches
  const accounts = await withBatch(
    addresses.map(addr => () => fetchAccount(addr))
  );
  
  // Batch all balance fetches
  const balances = await withBatch(
    addresses.map(addr => () => fetchBalance(addr))
  );
  
  return addresses.map((addr, i) => ({
    address: addr,
    account: accounts[i],
    balance: balances[i]
  }));
}
```

## Performance Considerations

### Batching Trade-offs

- **Pros**: Fewer network calls, better throughput
- **Cons**: Increased latency for first request in batch
- **Recommendation**: Use 50-100ms batch window for UI operations

### Deduplication Memory

- Cache size grows with unique requests
- Implement LRU eviction for long-running apps
- Clear cache on context changes (auth, network switch)

### Retry Overhead

- Failed requests increase total operation time
- Set reasonable maxRetries for user-facing operations
- Use longer delays for background operations

## Related Files

These utilities are implemented in:
- `src/sdk/utils/promise/withBatch.ts` - Request batching logic
- `src/sdk/utils/promise/withDedupe.ts` - Deduplication cache
- `src/sdk/utils/promise/withRetry.ts` - Retry with exponential backoff

For usage in the SDK, see:
- RPC client implementations
- Transaction broadcasting utilities
- Account and balance fetching functions
