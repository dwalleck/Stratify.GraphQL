# Stratify.GraphQL - Performance Guide

## Overview

This guide details the performance optimization strategies used throughout Stratify.GraphQL to achieve our goals of 50% faster cold start and 30% lower memory usage compared to reflection-based GraphQL libraries.

## Key Performance Principles

### 1. Zero Runtime Reflection

All type discovery and schema building happens at compile time through source generators.

### 2. Minimal Allocations

Hot paths are designed to minimize or eliminate heap allocations.

### 3. Cache-Friendly Code

Data structures and algorithms optimized for CPU cache efficiency.

### 4. Direct Execution

No indirection through delegates or expression trees in hot paths.

## High-Performance Data Structures

### System.Collections.Frozen (.NET 8+)

```csharp
// For static field resolver lookups
private static readonly FrozenDictionary<string, FieldResolver> FieldResolvers = 
    new Dictionary<string, FieldResolver>
    {
        ["id"] = ResolveId,
        ["name"] = ResolveName,
        ["email"] = ResolveEmail
    }.ToFrozenDictionary();

// 40% faster lookups than Dictionary
// Optimized for read-heavy scenarios
```

### System.Collections.Concurrent

```csharp
// For thread-safe caching
private static readonly ConcurrentDictionary<string, CompiledQuery> QueryCache = new();

// For DataLoader coordination
private static readonly ConcurrentDictionary<BatchKey, TaskCompletionSource<object[]>> PendingBatches = new();
```

### ArrayPool and Memory Pooling

```csharp
public sealed class ResolverContextPool
{
    private readonly ObjectPool<ResolverContext> _pool = new DefaultObjectPool<ResolverContext>(
        new ResolverContextPoolPolicy(), 
        Environment.ProcessorCount * 2);
    
    public ResolverContext Rent()
    {
        var context = _pool.Get();
        context.Reset();
        return context;
    }
    
    public void Return(ResolverContext context)
    {
        context.Clear(); // Clear references to avoid leaks
        _pool.Return(context);
    }
}
```

## Memory Management Strategies

### Span-Based Operations

```csharp
// String operations without allocation
public static bool IsFieldSelected(ReadOnlySpan<char> query, ReadOnlySpan<char> fieldName)
{
    // Search directly in the query span
    int index = query.IndexOf(fieldName);
    if (index == -1) return false;
    
    // Check word boundaries without substring
    bool validStart = index == 0 || !char.IsLetterOrDigit(query[index - 1]);
    bool validEnd = index + fieldName.Length >= query.Length || 
                    !char.IsLetterOrDigit(query[index + fieldName.Length]);
    
    return validStart && validEnd;
}
```

### Struct-Based Value Types

```csharp
// Small, frequently-used data as structs
public readonly struct FieldSelection
{
    public readonly int FieldId;
    public readonly int ParentId;
    public readonly SelectionFlags Flags;
    
    // Fits in 12 bytes, stays on stack
    public FieldSelection(int fieldId, int parentId, SelectionFlags flags)
    {
        FieldId = fieldId;
        ParentId = parentId;
        Flags = flags;
    }
}
```

### Pre-Sized Collections

```csharp
// Avoid resize allocations
public static Dictionary<string, object?> CreateResultDictionary(int fieldCount)
{
    // Pre-size to avoid rehashing
    return new Dictionary<string, object?>(fieldCount);
}
```

## Generated Code Optimizations

### Switch Expressions Over Lookups

```csharp
// Generated resolver uses switch expression
public static object? ResolveField(string fieldName, Book source) => fieldName switch
{
    "id" => source.Id,
    "title" => source.Title,
    "author" => source.Author,
    "isbn" when source.HasIsbn => source.Isbn,
    "publishedDate" => source.PublishedDate?.ToString("yyyy-MM-dd"),
    _ => throw new UnknownFieldException(fieldName)
};
// JIT compiles to jump table - O(1) performance
```

### Compile-Time Constants

```csharp
// Field name hashes computed at build time
public static class FieldHashes
{
    public const uint Id = 0x12345678u;
    public const uint Title = 0x87654321u;
    public const uint Author = 0xABCDEF01u;
}

// Ultra-fast field lookup
public static object? ResolveFieldByHash(uint hash, Book source) => hash switch
{
    FieldHashes.Id => source.Id,
    FieldHashes.Title => source.Title,
    FieldHashes.Author => source.Author,
    _ => null
};
```

### Inlined Hot Paths

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static bool IsNonNull(SelectionFlags flags) 
    => (flags & SelectionFlags.NonNull) != 0;

[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static void ValidateNonNull(object? value, string fieldName)
{
    if (value is null)
        ThrowNullFieldException(fieldName);
}

[DoesNotReturn]
private static void ThrowNullFieldException(string fieldName)
    => throw new GraphQLException($"Non-null field '{fieldName}' returned null");
```

## Query Execution Optimization

### Execution Plan Caching

```csharp
public sealed class QueryPlanCache
{
    private readonly ConcurrentDictionary<string, ExecutionPlan> _cache = new();
    
    public ExecutionPlan GetOrCreate(string query, Func<ExecutionPlan> factory)
    {
        // Fast path for cached plans
        if (_cache.TryGetValue(query, out var plan))
            return plan;
        
        // Slow path: create and cache
        plan = factory();
        _cache.TryAdd(query, plan);
        return plan;
    }
}
```

### Parallel Field Resolution

```csharp
// Generated code for parallel execution
public static async Task<Dictionary<string, object?>> ResolveFieldsParallel(
    Book source,
    FieldSelection[] selections,
    IServiceProvider services)
{
    var tasks = new Task<(string key, object? value)>[selections.Length];
    
    // Start all async operations
    for (int i = 0; i < selections.Length; i++)
    {
        var selection = selections[i];
        tasks[i] = ResolveFieldAsync(source, selection, services);
    }
    
    // Await all and build result
    var results = await Task.WhenAll(tasks);
    var dict = new Dictionary<string, object?>(selections.Length);
    
    foreach (var (key, value) in results)
    {
        dict[key] = value;
    }
    
    return dict;
}
```

## DataLoader Optimization

### Zero-Allocation Batching

```csharp
public sealed class BatchDataLoader<TKey, TValue> 
    where TKey : notnull
{
    private readonly Channel<LoadRequest> _channel;
    private readonly ArrayPool<TKey> _keyPool;
    
    public async ValueTask<TValue> LoadAsync(TKey key)
    {
        // Rent request from pool
        var request = LoadRequestPool<TKey, TValue>.Rent();
        request.Key = key;
        
        // Queue without allocation
        await _channel.Writer.WriteAsync(request);
        
        // Await result
        return await request.TaskCompletionSource.Task;
    }
    
    private async Task ProcessBatch()
    {
        var requests = new List<LoadRequest>(32); // Pre-sized
        
        // Drain channel
        while (_channel.Reader.TryRead(out var request))
        {
            requests.Add(request);
        }
        
        // Extract keys without allocation
        var keys = _keyPool.Rent(requests.Count);
        try
        {
            for (int i = 0; i < requests.Count; i++)
            {
                keys[i] = requests[i].Key;
            }
            
            // Batch fetch
            var results = await FetchBatch(keys.AsSpan(0, requests.Count));
            
            // Complete requests
            for (int i = 0; i < requests.Count; i++)
            {
                requests[i].TaskCompletionSource.SetResult(results[i]);
                LoadRequestPool<TKey, TValue>.Return(requests[i]);
            }
        }
        finally
        {
            _keyPool.Return(keys);
        }
    }
}
```

## Benchmarking Methodology

### Benchmark Setup

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class GraphQLBenchmarks
{
    private HotChocolateExecutor _hotChocolate;
    private StratifyExecutor _stratify;
    private string _simpleQuery;
    private string _complexQuery;
    
    [GlobalSetup]
    public void Setup()
    {
        _simpleQuery = "{ books { id title } }";
        _complexQuery = @"
            { 
                books(first: 100) { 
                    id 
                    title 
                    author { 
                        name 
                        email 
                        books { title } 
                    } 
                } 
            }";
    }
    
    [Benchmark(Baseline = true)]
    public Task<string> HotChocolate_Simple() 
        => _hotChocolate.ExecuteAsync(_simpleQuery);
    
    [Benchmark]
    public Task<string> Stratify_Simple() 
        => _stratify.ExecuteAsync(_simpleQuery);
}
```

### Key Metrics to Track

1. **Cold Start Time**: First request after app start
2. **Warm Request Time**: Subsequent requests
3. **Memory Allocations**: Bytes allocated per request
4. **GC Collections**: Gen 0/1/2 collections
5. **Working Set**: Total memory usage

### Performance Targets

- Cold Start: <50ms for 100-field schema
- Warm Request: <1ms overhead for simple queries
- Allocations: <1KB per simple query
- Throughput: >10,000 req/sec on single core

## Production Monitoring

### OpenTelemetry Integration

```csharp
private static readonly Meter Meter = new("Stratify.GraphQL");
private static readonly Counter<int> RequestCounter = 
    Meter.CreateCounter<int>("graphql.requests");
private static readonly Histogram<double> RequestDuration = 
    Meter.CreateHistogram<double>("graphql.request.duration");

public async Task<ExecutionResult> ExecuteAsync(QueryContext context)
{
    using var activity = Activity.StartActivity("GraphQL.Execute");
    var stopwatch = ValueStopwatch.StartNew();
    
    try
    {
        var result = await ExecuteCore(context);
        RequestCounter.Add(1, new("operation", context.OperationName));
        return result;
    }
    finally
    {
        RequestDuration.Record(
            stopwatch.GetElapsedTime().TotalMilliseconds,
            new("operation", context.OperationName));
    }
}
```

### Performance Alerts

- Cold start >100ms
- Request allocation >10KB
- GC Gen2 collection during request
- Query execution >100ms

## Optimization Checklist

### For Every Hot Path

- [ ] Zero allocations verified with benchmarks
- [ ] No boxing of value types
- [ ] Strings interned or avoided
- [ ] Collections pre-sized
- [ ] Async state machines minimized

### For Generated Code

- [ ] Switch expressions for lookups
- [ ] Constants inlined
- [ ] Methods marked for inlining
- [ ] Minimal indirection
- [ ] Cache-friendly data layout

### For Data Structures

- [ ] Frozen collections for static data
- [ ] Object pools for transient objects
- [ ] Structs for small value types
- [ ] Arrays over lists where possible
- [ ] Span-based APIs exposed
