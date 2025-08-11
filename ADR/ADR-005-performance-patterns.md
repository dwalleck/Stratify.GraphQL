# ADR-005: Performance-First Design Patterns

## Status

Accepted

## Context

To achieve our performance goals (50% faster cold start, 30% less memory), we need to make specific architectural choices around data structures, memory management, and code generation patterns.

## Decision

We will adopt these performance-first patterns throughout the implementation:

### 1. Zero-Allocation Hot Paths

- Use `Span<T>` and `ReadOnlySpan<T>` for temporary data
- Object pooling for frequently allocated objects
- Struct-based value types for small data

### 2. Compile-Time Optimization

- Switch expressions instead of dictionary lookups
- Direct method calls instead of delegates
- Inlined constants instead of field lookups

### 3. Cache-Friendly Data Structures

- `FrozenDictionary` for static lookups
- Struct arrays for execution plans
- Minimize pointer chasing

### 4. Memory Management

- ArrayPool for temporary buffers
- Pre-sized collections where possible
- Aggressive object reuse

## Consequences

### Positive

- **Performance**: Measurable improvements in all metrics
- **Predictability**: Consistent performance characteristics
- **Memory**: Lower GC pressure and working set

### Negative

- **Complexity**: More complex code in hot paths
- **Readability**: Performance optimizations can hurt clarity
- **Maintenance**: Requires performance expertise

### Neutral

- **Testing**: Need comprehensive performance benchmarks
- **Documentation**: Must document why code is "weird"

## Example Patterns

### Dictionary Lookup → Switch Expression

```csharp
// Before: Dictionary lookup (allocation + indirection)
_resolvers[fieldName](context);

// After: Switch expression (zero allocation, direct jump)
fieldName switch {
    "title" => ResolveTitle(context),
    "author" => ResolveAuthor(context),
    _ => throw new UnknownFieldException(fieldName)
};
```

### Object Allocation → Object Pool

```csharp
// Before: New allocation per request
var context = new ResolverContext(...);

// After: Pooled object reuse
var context = ResolverContextPool.Rent();
try { /* use context */ }
finally { ResolverContextPool.Return(context); }
```

## Monitoring

All performance patterns must be:

- Benchmarked before/after
- Monitored in production
- Documented with rationale
