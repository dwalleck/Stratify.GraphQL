# Stratify.GraphQL - Technical Design Document (TDD)

## 1. System Overview

### Architecture Overview

Stratify.GraphQL is a source generator-based GraphQL implementation that transforms compile-time type definitions into highly optimized runtime execution code. The system consists of:

- **Source Generator**: Analyzes C# types and generates GraphQL schema and execution code
- **Runtime Engine**: Minimal runtime that orchestrates generated code execution
- **ASP.NET Core Integration**: Middleware and service registration
- **Type System**: Compile-time type mapping and validation

### Component Interaction

```
[User Code with Attributes] → [Source Generator] → [Generated Code]
                                                           ↓
[GraphQL Query] → [Parser] → [Execution Engine] → [Generated Resolvers] → [Response]
```

### Key Design Decisions

- **Zero Runtime Reflection**: All type discovery happens at compile time
- **Direct Method Invocation**: No dynamic dispatch or expression trees
- **Minimal Allocations**: Object pooling and span-based operations throughout

## 2. Design Goals & Constraints

### Technical Goals

1. **Performance First**: Every design decision prioritizes runtime performance
2. **AOT Compatible**: Full ahead-of-time compilation support
3. **Debuggable**: Generated code must be readable and debuggable
4. **Type Safe**: Compile-time validation of GraphQL schemas
5. **Incremental**: Support incremental compilation for fast rebuilds

### Constraints

1. **No Runtime Schema Modification**: Schema must be fixed at compile time
2. **C# 12+ Required**: Latest language features for optimal code generation
3. **.NET 8+ Required**: Modern runtime features and source generator capabilities
4. **Limited Dynamic Scenarios**: Trade flexibility for performance

### Non-Goals

- Supporting every GraphQL edge case
- Runtime schema manipulation
- JavaScript/TypeScript compatibility
- Backward compatibility with older .NET versions

## 3. Architecture Design

### Source Generator Architecture

#### Incremental Source Generator Pipeline

```csharp
[Generator]
public class GraphQLSourceGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Stage 1: Collect GraphQL type definitions
        var typeProvider = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: IsGraphQLType,
                transform: GetTypeModel
            )
            .Where(type => type != null);
        
        // Stage 2: Build schema model
        var schemaModel = typeProvider
            .Collect()
            .Select(BuildSchemaModel);
        
        // Stage 3: Generate code
        context.RegisterSourceOutput(schemaModel, GenerateCode);
    }
}
```

#### Code Generation Strategy

- **Template-Based Generation**: Use T4-like templates for complex code
- **StringBuilder for Simple Code**: Direct string building for simple patterns
- **Roslyn SyntaxFactory**: For complex AST manipulation

### Runtime Architecture

#### Execution Engine Design

```csharp
public sealed class GraphQLExecutor
{
    private readonly Dictionary<string, IGeneratedQueryExecutor> _executors;
    private readonly GraphQLDocumentParser _parser;
    private readonly IServiceProvider _services;
    
    public async ValueTask<ExecutionResult> ExecuteAsync(
        string query,
        Variables? variables = null,
        CancellationToken cancellationToken = default)
    {
        // Parse query (cached)
        var document = _parser.Parse(query);
        
        // Get generated executor
        var executor = _executors[document.GetQueryHash()];
        
        // Execute with zero allocations
        return await executor.ExecuteAsync(
            _services,
            variables,
            cancellationToken);
    }
}
```

#### Query Feature Support

```csharp
// Variable handling
public readonly struct Variables
{
    private readonly FrozenDictionary<string, object?> _values;
    
    public T? GetValue<T>(string name) => 
        _values.TryGetValue(name, out var value) ? (T?)value : default;
}

// Alias resolution
public sealed class FieldSelection
{
    public string Name { get; init; }
    public string? Alias { get; init; }
    public string ResponseKey => Alias ?? Name;
}

// Fragment support
public abstract class FragmentDefinition
{
    public string Name { get; init; }
    public string TypeCondition { get; init; }
    public abstract bool Matches(GraphQLType type);
}
```

### Type System Design

#### GraphQL to C# Type Mapping

```csharp
// Source generator discovers this pattern
[GraphQLObject]
public partial class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    [GraphQLField]
    public async Task<Category> GetCategory(
        [Service] ICategoryService service)
        => await service.GetByProductIdAsync(Id);
}

// Generates efficient resolver
partial class Product : IGraphQLObject
{
    public static void ResolveFields(
        IFieldResolverMap map,
        Product source,
        IResolverContext context)
    {
        // Direct property access, no reflection
        map.Add("id", source.Id);
        map.Add("name", source.Name);
        map.Add("price", source.Price);
        
        // Async field with service injection
        map.AddAsync("category", async ctx =>
        {
            var service = ctx.GetService<ICategoryService>();
            return await source.GetCategory(service);
        });
    }
}
```

## 4. Key Performance Principles

### Memory Management

#### Allocation Reduction Strategies

1. **Span-Based Operations**

   ```csharp
   // Use Span<T> for temporary data manipulation
   public readonly struct FieldName
   {
       private readonly ReadOnlySpan<char> _name;
       
       public bool Equals(ReadOnlySpan<char> other)
           => _name.SequenceEqual(other);
   }
   ```

2. **Object Pooling**

   ```csharp
   // Pool frequently allocated objects
   public sealed class ResolverContextPool
   {
       private readonly ObjectPool<ResolverContext> _pool;
       
       public ResolverContext Rent() => _pool.Get();
       public void Return(ResolverContext context) => _pool.Return(context);
   }
   ```

3. **Struct-Based Value Types**

   ```csharp
   // Small, immutable data as structs
   public readonly struct FieldSelection
   {
       public readonly int FieldId;
       public readonly SelectionFlags Flags;
       public readonly IReadOnlyList<FieldSelection> Children;
   }
   ```

### Cache-Friendly Patterns

1. **Data Locality**

   ```csharp
   // Group related data together
   public struct ExecutionNode
   {
       public int TypeId;
       public int FieldId;
       public int ParentIndex;
       public ExecutionFlags Flags;
       // All data fits in single cache line
   }
   ```

2. **Frozen Collections**

   ```csharp
   // Use FrozenDictionary for static lookups
   private static readonly FrozenDictionary<string, FieldResolver> FieldResolvers =
       new Dictionary<string, FieldResolver>
       {
           ["id"] = ResolveId,
           ["name"] = ResolveName,
           ["price"] = ResolvePrice
       }.ToFrozenDictionary();
   ```

### Generated Code Efficiency

1. **Switch Expressions Over Dictionary Lookups**

   ```csharp
   // Generate this instead of dictionary lookups
   public static ResolverDelegate GetResolver(string fieldName) => fieldName switch
   {
       "Book.title" => ResolveBookTitle,
       "Book.author" => ResolveBookAuthor,
       "Author.name" => ResolveAuthorName,
       _ => throw new InvalidOperationException($"Unknown field: {fieldName}")
   };
   
   // Instead of:
   // private static readonly Dictionary<string, ResolverDelegate> _resolvers = ...
   ```

2. **Compile-Time Constants with Hashing**

   ```csharp
   // Generate constants instead of runtime lookups
   public static class FieldHashes
   {
       public const uint BookTitle = 0x12345678u;    // Computed at build time
       public const uint BookAuthor = 0x87654321u;
       public const uint AuthorName = 0xABCDEF00u;
   }
   
   // Use in hot paths - JIT optimizes this to a jump table
   public static ResolverDelegate GetResolverByHash(uint hash) => hash switch
   {
       FieldHashes.BookTitle => ResolveBookTitle,
       FieldHashes.BookAuthor => ResolveBookAuthor,
       FieldHashes.AuthorName => ResolveAuthorName,
       _ => throw new InvalidOperationException($"Unknown field hash: {hash:X}")
   };
   ```

3. **Direct Method Calls Without Delegates**

   ```csharp
   // No delegates or dynamic dispatch in hot paths
   public static async ValueTask<Product> GetProduct(
       IProductService service,
       int id)
       => await service.GetByIdAsync(id);
   ```

4. **Inlined Field Resolution**

   ```csharp
   // Generate specific resolvers for each type
   public static object ResolveProductFields(string fieldName, Product source)
       => fieldName switch
       {
           "id" => source.Id,
           "name" => source.Name,
           "price" => source.Price,
           "category" => LoadCategory(source.CategoryId), // Lazy load reference
           _ => throw new UnknownFieldException(fieldName)
       };
   ```

## 5. Design Patterns Implementation

### Template Method Pattern for Type Generation

```csharp
public abstract class GraphQLTypeGenerator<T>
{
    // Template method - structure is fixed
    public string GenerateType()
    {
        var builder = new StringBuilder();
        
        builder.AppendLine(GenerateHeader());
        builder.AppendLine(GenerateFields());
        builder.AppendLine(GenerateResolvers());
        builder.AppendLine(GenerateFooter());
        
        return builder.ToString();
    }
    
    // Source generator implements these
    protected abstract string GenerateHeader();
    protected abstract string GenerateFields();
    protected abstract string GenerateResolvers();
    protected abstract string GenerateFooter();
}
```

### Strategy Pattern for Query Execution

```csharp
// Different strategies for different query types
public interface IQueryExecutionStrategy
{
    ValueTask<ExecutionResult> ExecuteAsync(
        ExecutionContext context,
        CancellationToken cancellationToken);
}

// Generated strategies optimized for specific patterns
public sealed class SimpleQueryStrategy : IQueryExecutionStrategy
{
    // Optimized for queries without fragments or variables
}

public sealed class ComplexQueryStrategy : IQueryExecutionStrategy
{
    // Handles fragments, variables, directives
}
```

### Registry Pattern for Type Resolution

```csharp
// Compile-time registry with zero runtime cost
public static class GeneratedTypeRegistry
{
    // Generated at compile time
    private static readonly FrozenDictionary<Type, GraphQLTypeInfo> TypeMap =
        new Dictionary<Type, GraphQLTypeInfo>
        {
            [typeof(Product)] = new GraphQLTypeInfo("Product", ProductResolvers),
            [typeof(Category)] = new GraphQLTypeInfo("Category", CategoryResolvers),
            // ... more types
        }.ToFrozenDictionary();
    
    public static GraphQLTypeInfo GetTypeInfo(Type type)
        => TypeMap[type];
}
```

## 6. API Design

### Public API Surface

#### Type System Foundation

```csharp
// Object Types
[GraphQLObject]
public partial class BookType : GraphQLObjectType<Book>
{
    [GraphQLField("The book's title")]
    public string Title => Source.Title;
    
    [GraphQLField]
    public async Task<Author> Author([Service] IAuthorService service) 
        => await service.GetByIdAsync(Source.AuthorId);
    
    [GraphQLField(Name = "discountedPrice", Description = "Price after discount")]
    public decimal GetDiscountedPrice(
        [GraphQLArgument(Description = "Discount percentage")] float discount)
        => Source.Price * (1 - discount / 100);
}

// Input Types
[GraphQLInputType]
public partial class CreateBookInput : GraphQLInputType<CreateBookRequest>
{
    [GraphQLField] public string Title { get; set; }
    [GraphQLField] public string AuthorId { get; set; }
    [GraphQLField] public DateTime? PublishDate { get; set; }
}

// Enums
[GraphQLEnum]
public enum BookStatus
{
    [GraphQLValue("DRAFT")] Draft,
    [GraphQLValue("PUBLISHED")] Published,
    [GraphQLValue("ARCHIVED")] Archived
}

// Interfaces
[GraphQLInterface]
public partial interface IContentType
{
    [GraphQLField] string Title { get; }
    [GraphQLField] DateTime CreatedAt { get; }
}

// Unions
[GraphQLUnion(typeof(BookType), typeof(ArticleType))]
public partial class SearchResult { }

// Schema Root
[GraphQLSchema]
public class BookStoreSchema
{
    [Query]
    public IQueryable<Book> Books([Service] IBookService service)
        => service.GetBooks();
    
    [Mutation]
    public async Task<Book> CreateBook(
        CreateBookInput input,
        [Service] IBookService service)
        => await service.CreateAsync(input);
}
```

#### Service Registration

```csharp
// Simple, convention-based registration
builder.Services.AddGraphQL<MySchema>(options =>
{
    options.MaxQueryDepth = 10;
    options.MaxQueryComplexity = 1000;
    options.EnableIntrospection = !env.IsProduction();
});

// Minimal middleware setup
app.MapGraphQL("/graphql");
```

### Extension Points

#### Custom Scalar Types

```csharp
[GraphQLScalar]
public readonly struct Money : IGraphQLScalar<decimal>
{
    public static Money Parse(string value) => new(decimal.Parse(value));
    public string Serialize() => Value.ToString("C");
}
```

#### Introspection Support

```csharp
// Generated introspection data (compile-time, not dynamic)
public static partial class IntrospectionData
{
    public static readonly __Schema Schema = new()
    {
        Types = new[]
        {
            new __Type { Name = "Book", Kind = TypeKind.OBJECT, Fields = BookFields },
            new __Type { Name = "Author", Kind = TypeKind.OBJECT, Fields = AuthorFields },
            new __Type { Name = "BookStatus", Kind = TypeKind.ENUM, EnumValues = BookStatusValues },
            // All types known at compile time
        },
        QueryType = new __Type { Name = "Query" },
        MutationType = new __Type { Name = "Mutation" }
    };
    
    // Generated field definitions
    private static readonly __Field[] BookFields = new[]
    {
        new __Field { Name = "title", Type = new __Type { Name = "String", Kind = TypeKind.SCALAR } },
        new __Field { Name = "author", Type = new __Type { Name = "Author", Kind = TypeKind.OBJECT } }
    };
}
```

#### Field Middleware

```csharp
[GraphQLMiddleware]
public class TimingMiddleware : IFieldMiddleware
{
    public async ValueTask<object?> InvokeAsync(
        FieldMiddlewareContext context,
        FieldDelegate next)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            return await next(context);
        }
        finally
        {
            context.SetExtension("duration", sw.ElapsedMilliseconds);
        }
    }
}
```

## 7. Federation Architecture

### Build-Time Federation

```csharp
// Service references known at compile time
[GraphQLFederatedSchema]
public class GatewaySchema
{
    [FederatedService("products")]
    public IProductService Products { get; set; }
    
    [FederatedService("inventory")]
    public IInventoryService Inventory { get; set; }
}

// Source generator creates efficient query planner
public sealed class GeneratedQueryPlanner
{
    public QueryPlan PlanQuery(DocumentNode query)
    {
        // Static analysis of query
        // Optimal execution plan generation
        // No runtime service discovery
    }
}
```

## 8. Developer Experience Design

### Compile-Time Validation

```csharp
// Source generator produces descriptive compile-time errors
public class GraphQLDiagnostics
{
    public static readonly DiagnosticDescriptor FieldNotFound = new(
        "GQL001",
        "Field references non-existent property",
        "Field '{0}' references '{1}' which doesn't exist on type '{2}'. Available: {3}",
        "GraphQL.TypeSystem",
        DiagnosticSeverity.Error,
        isEnabledByDefault: true);
    
    public static readonly DiagnosticDescriptor CircularDependency = new(
        "GQL002",
        "Circular dependency detected in GraphQL types",
        "Circular dependency: {0}",
        "GraphQL.TypeSystem",
        DiagnosticSeverity.Error,
        isEnabledByDefault: true);
}

// Usage in source generator
context.ReportDiagnostic(Diagnostic.Create(
    GraphQLDiagnostics.FieldNotFound,
    location,
    fieldName,
    propertyPath,
    typeName,
    string.Join(", ", availableProperties)));
```

### Debuggable Generated Code

```csharp
// Generated with #line directives for source mapping
#line 1 "BookType.cs"
public static partial class BookType_Resolvers
{
    /// <summary>
    /// Resolver for field 'title' on type 'Book'
    /// Source: BookType.cs:42 - Property Title
    /// </summary>
    [DebuggerStepThrough] // Skip unless explicitly stepping into
    public static async Task<object> ResolveTitle(
        Book source, 
        IResolverContext context)
    {
        #line 42 "BookType.cs"
        try
        {
            return source.Title; // F12 navigates to original property
        }
        catch (Exception ex)
        {
            context.ReportError(new GraphQLError
            {
                Message = ex.Message,
                Path = context.Path,
                Code = "FIELD_ERROR",
                Extensions = new Dictionary<string, object>
                {
                    ["field"] = "title",
                    ["type"] = "Book"
                }
            });
            return null;
        }
        #line default
    }
}
```

### Hot Reload Support

```csharp
// Metadata preservation for hot reload
[assembly: MetadataUpdateHandler(typeof(GraphQLSchemaUpdateHandler))]

public static class GraphQLSchemaUpdateHandler
{
    public static void ClearCache(Type[]? updatedTypes)
    {
        // Called by hot reload infrastructure
        if (updatedTypes?.Any(t => t.HasAttribute<GraphQLObjectAttribute>()) == true)
        {
            SchemaCache.Clear();
            ResolverCache.Clear();
        }
    }
    
    public static void UpdateApplication(Type[]? updatedTypes)
    {
        // Recompile affected schema parts
        foreach (var type in updatedTypes ?? Array.Empty<Type>())
        {
            if (type.HasAttribute<GraphQLObjectAttribute>())
            {
                RegenerateTypeResolvers(type);
            }
        }
    }
}
```

### Rich Error Messages

```csharp
// Structured error information with helpful context
public readonly struct GraphQLErrorInfo
{
    public string Message { get; init; }
    public string[] Path { get; init; }
    public Location[] Locations { get; init; }
    public ErrorCode Code { get; init; }
    public IReadOnlyDictionary<string, object?> Extensions { get; init; }
    
    // Helper for creating user-friendly errors
    public static GraphQLErrorInfo FieldNotFound(
        string fieldName,
        string typeName,
        string[] availableFields)
    {
        return new GraphQLErrorInfo
        {
            Message = $"Field '{fieldName}' not found on type '{typeName}'",
            Code = ErrorCode.FieldNotFound,
            Extensions = new Dictionary<string, object?>
            {
                ["field"] = fieldName,
                ["type"] = typeName,
                ["suggestions"] = FindSimilarFields(fieldName, availableFields)
            }
        };
    }
}
```

### IntelliSense Integration

```csharp
// Generated XML documentation for IDE support
/// <summary>
/// GraphQL type 'Book' mapped from <see cref="Models.Book"/>
/// </summary>
/// <remarks>
/// Available fields:
/// - title: String!
/// - author: Author
/// - publishedDate: DateTime
/// - isbn: String
/// </remarks>
[GeneratedCode("Stratify.GraphQL.Generator", "1.0.0")]
public static class BookType
{
    // Field resolvers with full IntelliSense
}
```

## 9. Performance Optimizations

### DataLoader Implementation

```csharp
// Zero-allocation batch loading
public sealed class BatchDataLoader<TKey, TValue>
    where TKey : notnull
{
    private readonly Channel<LoadRequest> _channel;
    private readonly IDataLoaderFetch<TKey, TValue> _fetch;
    
    public async ValueTask<TValue> LoadAsync(TKey key)
    {
        // Rent from pool
        var request = LoadRequestPool.Rent();
        request.Key = key;
        
        await _channel.Writer.WriteAsync(request);
        return await request.Task;
    }
}
```

### Query Complexity Analysis

```csharp
// Compile-time complexity calculation where possible
[GraphQLField(Complexity = 10)]
public async Task<IEnumerable<Product>> SearchProducts(
    string query,
    [GraphQLArgument] int first = 10)
{
    // Complexity = 10 + (first * 1)
}
```

## 10. Testing Strategy

### Source Generator Testing

```csharp
// Snapshot testing for generated code
[Test]
public async Task GeneratesCorrectResolverCode()
{
    var source = """
        [GraphQLObject]
        public class Product
        {
            public int Id { get; set; }
            public string Name { get; set; }
        }
        """;
    
    var generated = await GenerateCode(source);
    await Verify(generated);
}
```

### Performance Benchmarks

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class GraphQLBenchmarks
{
    [Benchmark(Baseline = true)]
    public async Task<string> HotChocolate()
        => await _hotChocolate.ExecuteAsync(Query);
    
    [Benchmark]
    public async Task<string> StratifyGraphQL()
        => await _stratify.ExecuteAsync(Query);
}
```

## 11. Security Design

### Query Validation

- Depth limiting enforced at parse time
- Complexity calculation during execution planning
- Query whitelisting support for production

### Input Sanitization

- All inputs validated before processing
- Injection attack prevention
- Safe error messages (no internal details exposed)

## 12. Performance Monitoring Integration

### System.Diagnostics.DiagnosticSource

```csharp
// For ETW and EventSource integration
private static readonly DiagnosticSource _diagnosticSource = 
    new DiagnosticListener("GraphQL.Execution");

public async Task<object> ExecuteFieldAsync(IFieldContext context)
{
    if (_diagnosticSource.IsEnabled("GraphQL.Field.Start"))
    {
        _diagnosticSource.Write("GraphQL.Field.Start", 
            new { FieldName = context.Field.Name });
    }
    
    try
    {
        return await ExecuteFieldCore(context);
    }
    finally
    {
        if (_diagnosticSource.IsEnabled("GraphQL.Field.Stop"))
        {
            _diagnosticSource.Write("GraphQL.Field.Stop", 
                new { FieldName = context.Field.Name });
        }
    }
}
```

### System.Diagnostics.Metrics API

```csharp
// Modern metrics API for OpenTelemetry compatibility
private static readonly Meter _meter = new("Stratify.GraphQL");
private static readonly Counter<int> _requestCounter = 
    _meter.CreateCounter<int>("graphql.requests");
private static readonly Histogram<double> _requestDuration = 
    _meter.CreateHistogram<double>("graphql.request.duration");

// Usage in generated code
public async ValueTask<ExecutionResult> ExecuteAsync(QueryContext context)
{
    using var activity = Activity.StartActivity("GraphQL.Execute");
    var stopwatch = ValueStopwatch.StartNew();
    
    try
    {
        var result = await ExecuteCore(context);
        _requestCounter.Add(1, 
            new("operation", context.OperationName),
            new("type", context.OperationType.ToString()));
        return result;
    }
    finally
    {
        _requestDuration.Record(
            stopwatch.GetElapsedTime().TotalMilliseconds,
            new("operation", context.OperationName));
    }
}
```

## 13. Migration Strategy

### From HotChocolate

```csharp
// Migration analyzer identifies patterns
public class HotChocolateMigrationAnalyzer : DiagnosticAnalyzer
{
    // Suggests attribute replacements
    // Identifies incompatible patterns
    // Provides fix suggestions
}
```

### Gradual Migration Support

- Side-by-side operation during transition
- Schema comparison tools
- Performance comparison utilities

## 14. Technical Risks & Mitigations

### Risk: Source Generator Limitations

- **Mitigation**: Prototype complex scenarios early
- **Fallback**: Runtime code generation for edge cases

### Risk: Debugging Generated Code

- **Mitigation**: Readable code generation, source maps
- **Mitigation**: Comprehensive logging and diagnostics

### Risk: AOT Compatibility Issues

- **Mitigation**: Continuous AOT testing
- **Mitigation**: Work closely with .NET team

## 15. Future Considerations

### Potential Enhancements

1. **Subscription Support**: Using AsyncEnumerable and channels
2. **Schema Evolution**: Versioning strategies
3. **Distributed Tracing**: OpenTelemetry integration
4. **Advanced Caching**: Response caching, query plan caching

### Extensibility Points

- Custom execution strategies
- Pluggable type discovery
- Schema transformation pipeline
- Custom directive processing

## Appendices

### Appendix A: Performance Benchmarks

[Detailed benchmark results comparing with other GraphQL libraries]

### Appendix B: Generated Code Examples

[Sample generated code for various scenarios]

### Appendix C: Design Decision Records

[Detailed rationale for key technical decisions]
