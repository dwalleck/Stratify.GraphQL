# Stratify.GraphQL - Design Patterns Catalog

## Overview

This catalog documents the design patterns used in Stratify.GraphQL and explains how source generation enables us to "compile away" traditional runtime abstractions while maintaining their architectural benefits.

## Key Insight: Compiling Away Abstractions

Traditional design patterns often involve runtime polymorphism and dynamic dispatch. Our source generator approach generates specialized code at compile time, providing:

- Design benefits of well-known patterns
- Performance of direct, specialized code
- Zero runtime overhead

## Code Generation Patterns

### Template Method Pattern

Used for generating GraphQL type definitions with a fixed structure but varying details.

```csharp
// Base template for type generation
public abstract class GraphQLTypeGenerator<T>
{
    public string Generate()
    {
        var builder = new StringBuilder();
        
        // Fixed structure
        builder.AppendLine(GenerateHeader());
        builder.AppendLine(GenerateFields());
        builder.AppendLine(GenerateResolvers());
        builder.AppendLine(GenerateFooter());
        
        return builder.ToString();
    }
    
    // Varying parts implemented by source generator
    protected abstract string GenerateHeader();
    protected abstract string GenerateFields();
    protected abstract string GenerateResolvers();
    protected abstract string GenerateFooter();
}

// Generated implementation for specific type
public sealed class BookTypeGenerator : GraphQLTypeGenerator<Book>
{
    protected override string GenerateHeader()
        => "public static partial class BookType {";
    
    protected override string GenerateFields()
        => """
           public static readonly GraphQLField[] Fields = new[]
           {
               new GraphQLField("id", typeof(int)),
               new GraphQLField("title", typeof(string)),
               new GraphQLField("author", typeof(Author))
           };
           """;
    
    // ... other methods
}
```

### Builder Pattern

For constructing complex schema configurations at compile time.

```csharp
// Schema builder used by source generator
public class SchemaBuilder
{
    private readonly List<TypeDefinition> _types = new();
    private readonly List<FieldDefinition> _queries = new();
    private readonly List<FieldDefinition> _mutations = new();
    
    public SchemaBuilder AddType(TypeDefinition type)
    {
        _types.Add(type);
        return this;
    }
    
    public SchemaBuilder AddQuery(string name, MethodInfo method)
    {
        _queries.Add(new FieldDefinition(name, method));
        return this;
    }
    
    public GeneratedSchema Build()
    {
        // Generate optimized schema representation
        return new GeneratedSchema
        {
            TypeMap = _types.ToFrozenDictionary(t => t.Name),
            QueryFields = _queries.ToFrozenDictionary(f => f.Name),
            MutationFields = _mutations.ToFrozenDictionary(f => f.Name)
        };
    }
}
```

## Structural Patterns

### Registry Pattern

For efficient type and resolver lookup without reflection.

```csharp
// Generated type registry
public static class GraphQLTypeRegistry
{
    // Compile-time generated frozen dictionary
    private static readonly FrozenDictionary<Type, TypeMetadata> TypeMap = 
        new Dictionary<Type, TypeMetadata>
        {
            [typeof(Book)] = new TypeMetadata
            {
                GraphQLName = "Book",
                Fields = BookType.Fields,
                Resolver = BookType.CreateResolver
            },
            [typeof(Author)] = new TypeMetadata
            {
                GraphQLName = "Author",
                Fields = AuthorType.Fields,
                Resolver = AuthorType.CreateResolver
            }
        }.ToFrozenDictionary();
    
    // Zero-overhead lookup
    public static TypeMetadata GetMetadata(Type type)
        => TypeMap[type];
}

// Generated resolver registry with switch expression
public static class ResolverRegistry
{
    public static Func<object, IResolverContext, ValueTask<object?>> GetResolver(
        string typeName, 
        string fieldName)
        => (typeName, fieldName) switch
        {
            ("Book", "title") => BookResolvers.ResolveTitle,
            ("Book", "author") => BookResolvers.ResolveAuthor,
            ("Author", "name") => AuthorResolvers.ResolveName,
            ("Author", "books") => AuthorResolvers.ResolveBooks,
            _ => throw new UnknownFieldException(typeName, fieldName)
        };
}
```

### Facade Pattern

Simplified API over complex generated code.

```csharp
// User-facing simple API
public static class GraphQL
{
    public static IServiceCollection AddGraphQL<TSchema>(
        this IServiceCollection services,
        Action<GraphQLOptions>? configure = null)
    {
        // Hide complexity of:
        // - Source generator integration
        // - Type discovery
        // - Schema building
        // - Service registration
        
        var options = new GraphQLOptions();
        configure?.Invoke(options);
        
        // Register all generated components
        services.AddSingleton<IGraphQLSchema>(GeneratedSchema.Instance);
        services.AddSingleton<IQueryExecutor>(GeneratedExecutor.Instance);
        services.AddScoped<IResolverContext, ResolverContext>();
        
        return services;
    }
}
```

## Behavioral Patterns

### Strategy Pattern

Different execution strategies for different query types.

```csharp
// Interface for execution strategies
public interface IExecutionStrategy
{
    ValueTask<ExecutionResult> ExecuteAsync(
        QueryContext context,
        CancellationToken cancellationToken);
}

// Generated strategies optimized at compile time
public sealed class SimpleQueryStrategy : IExecutionStrategy
{
    // Optimized for queries without fragments or complex features
    public async ValueTask<ExecutionResult> ExecuteAsync(
        QueryContext context,
        CancellationToken cancellationToken)
    {
        // Direct field resolution without overhead
        var result = new Dictionary<string, object?>(context.FieldCount);
        
        foreach (var field in context.Fields)
        {
            result[field.ResponseKey] = field.FieldId switch
            {
                FieldIds.Book_Title => ((Book)context.Source).Title,
                FieldIds.Book_Author => await ResolveAuthor(context),
                _ => null
            };
        }
        
        return new ExecutionResult(result);
    }
}

public sealed class ComplexQueryStrategy : IExecutionStrategy
{
    // Handles fragments, directives, complex selections
    // More overhead but still optimized
}

// Strategy selection at execution time
public static IExecutionStrategy SelectStrategy(ParsedQuery query)
    => query.Complexity switch
    {
        QueryComplexity.Simple => SimpleQueryStrategy.Instance,
        QueryComplexity.WithFragments => FragmentQueryStrategy.Instance,
        QueryComplexity.WithDirectives => DirectiveQueryStrategy.Instance,
        _ => ComplexQueryStrategy.Instance
    };
```

### Command Pattern

Each field resolution as a pre-compiled command.

```csharp
// Generated field resolver commands
public static class BookFieldResolvers
{
    // Each resolver is a self-contained command
    public static readonly FieldResolver ResolveTitle = new(
        fieldName: "title",
        resolver: (source, context) => 
        {
            var book = (Book)source;
            return new ValueTask<object?>(book.Title);
        },
        complexity: 1,
        isAsync: false
    );
    
    public static readonly FieldResolver ResolveAuthor = new(
        fieldName: "author",
        resolver: async (source, context) =>
        {
            var book = (Book)source;
            var authorService = context.GetService<IAuthorService>();
            return await authorService.GetByIdAsync(book.AuthorId);
        },
        complexity: 10,
        isAsync: true
    );
}

// Command execution
public async ValueTask<object?> ExecuteFieldResolver(
    string fieldName,
    object source,
    IResolverContext context)
{
    var resolver = GetResolver(fieldName);
    return await resolver.Execute(source, context);
}
```

### Chain of Responsibility Pattern

For field middleware, but unrolled at compile time.

```csharp
// Traditional runtime chain
public interface IFieldMiddleware
{
    ValueTask<object?> InvokeAsync(
        IMiddlewareContext context,
        FieldDelegate next);
}

// Generated unrolled chain for specific field
public static async ValueTask<object?> ResolveBookTitleWithMiddleware(
    Book source,
    IResolverContext context)
{
    // Authorization check (inlined)
    if (!context.User.IsAuthenticated)
        throw new UnauthorizedException();
    
    // Timing start (inlined)
    var stopwatch = ValueStopwatch.StartNew();
    
    try
    {
        // Actual resolution (inlined)
        var result = source.Title;
        
        // Transform (inlined)
        if (context.Variables.GetValue<bool>("uppercase"))
            result = result?.ToUpper();
        
        return result;
    }
    finally
    {
        // Timing end (inlined)
        context.SetMetric("duration", stopwatch.GetElapsedTime());
    }
}
```

## Data Access Patterns

### Repository Pattern

Generated type-safe data access without reflection.

```csharp
// Generated repository for each type
public static partial class BookRepository
{
    // Type-safe query methods generated from GraphQL schema
    public static async Task<Book?> GetById(
        int id,
        IDbConnection connection)
    {
        const string sql = """
            SELECT Id, Title, AuthorId, PublishedDate
            FROM Books
            WHERE Id = @id
            """;
        
        return await connection.QuerySingleOrDefaultAsync<Book>(sql, new { id });
    }
    
    public static async Task<IEnumerable<Book>> GetByAuthor(
        int authorId,
        IDbConnection connection)
    {
        const string sql = """
            SELECT Id, Title, AuthorId, PublishedDate
            FROM Books
            WHERE AuthorId = @authorId
            ORDER BY PublishedDate DESC
            """;
        
        return await connection.QueryAsync<Book>(sql, new { authorId });
    }
}
```

### Unit of Work Pattern

For mutation transactions with generated coordination.

```csharp
// Generated unit of work for mutations
public sealed class GraphQLUnitOfWork : IAsyncDisposable
{
    private readonly IDbTransaction _transaction;
    private readonly List<Func<Task>> _operations = new();
    
    public void AddOperation<T>(T entity, Operation operation)
    {
        _operations.Add(operation switch
        {
            Operation.Insert => () => GeneratedOperations.Insert(entity, _transaction),
            Operation.Update => () => GeneratedOperations.Update(entity, _transaction),
            Operation.Delete => () => GeneratedOperations.Delete(entity, _transaction),
            _ => throw new InvalidOperationException()
        });
    }
    
    public async Task CommitAsync()
    {
        foreach (var operation in _operations)
        {
            await operation();
        }
        
        await _transaction.CommitAsync();
    }
}
```

## Caching Patterns

### Proxy Pattern for caching, but with inlined logic

```csharp
// Generated caching proxy
public static class CachedBookResolver
{
    private static readonly IMemoryCache Cache = new MemoryCache(new MemoryCacheOptions());
    
    public static async ValueTask<Book?> GetBook(
        int id,
        IBookService service)
    {
        // Cache key generation (inlined)
        var cacheKey = $"book:{id}";
        
        // Fast path - cache hit
        if (Cache.TryGetValue<Book>(cacheKey, out var cached))
            return cached;
        
        // Slow path - fetch and cache
        var book = await service.GetByIdAsync(id);
        if (book != null)
        {
            Cache.Set(cacheKey, book, TimeSpan.FromMinutes(5));
        }
        
        return book;
    }
}
```

## Performance Benefits

### Compile-Time Optimizations

1. **Virtual dispatch elimination**: Direct method calls instead of interfaces
2. **Inlining opportunities**: JIT can inline generated methods
3. **Dead code elimination**: Unused code paths removed
4. **Constant folding**: Known values computed at compile time

### Runtime Benefits

1. **Zero allocation patterns**: No boxing, minimal object creation
2. **Cache-friendly code**: Sequential memory access patterns
3. **Predictable performance**: No reflection or dynamic dispatch
4. **Reduced indirection**: Direct field access and method calls

## Pattern Selection Guidelines

### Use Template Method When

- Structure is fixed but details vary
- Need consistent code generation
- Want to enforce architectural patterns

### Use Strategy When

- Different algorithms for different scenarios
- Performance characteristics vary by case
- Runtime selection based on query analysis

### Use Registry When

- Need fast lookups without reflection
- Type set is known at compile time
- Performance is critical

### Use Command When

- Operations are self-contained
- Need to compose operations dynamically
- Want to add metadata to operations

## Anti-Patterns to Avoid

### Runtime Type Discovery

```csharp
// ❌ Don't do this
var types = Assembly.GetExecutingAssembly()
    .GetTypes()
    .Where(t => t.HasAttribute<GraphQLObject>());

// ✅ Do this (generated)
var types = GraphQLTypes.All;
```

### Dynamic Delegate Creation

```csharp
// ❌ Don't do this
var resolver = Expression.Lambda<Func<object, object>>(
    Expression.Property(parameter, propertyInfo)).Compile();

// ✅ Do this (generated)
public static object ResolveTitle(object source) 
    => ((Book)source).Title;
```

### Reflection-Based Validation

```csharp
// ❌ Don't do this
var properties = type.GetProperties()
    .Where(p => p.HasAttribute<Required>());

// ✅ Do this (generated)
public static ValidationResult Validate(Book book)
{
    if (string.IsNullOrEmpty(book.Title))
        return ValidationResult.Error("Title is required");
    
    return ValidationResult.Success;
}
```
