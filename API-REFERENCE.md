# Stratify.GraphQL - API Reference

## Table of Contents
1. [Type Definition Attributes](#type-definition-attributes)
2. [Schema Configuration](#schema-configuration)
3. [Field Configuration](#field-configuration)
4. [Service Registration](#service-registration)
5. [Advanced Features](#advanced-features)
6. [Extension Points](#extension-points)

## Type Definition Attributes

### `[GraphQLObject]`
Marks a class as a GraphQL object type.

```csharp
[GraphQLObject]
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; }
}

// With custom name
[GraphQLObject(Name = "BookType", Description = "Represents a book in the system")]
public class Book { }
```

### `[GraphQLInputType]`
Marks a class as a GraphQL input type for mutations.

```csharp
[GraphQLInputType]
public class CreateBookInput
{
    [GraphQLField(Required = true)]
    public string Title { get; set; }
    
    [GraphQLField]
    public string? Description { get; set; }
    
    [GraphQLField(DefaultValue = "DRAFT")]
    public BookStatus Status { get; set; }
}
```

### `[GraphQLInterface]`
Defines a GraphQL interface type.

```csharp
[GraphQLInterface]
public interface INode
{
    [GraphQLField]
    string Id { get; }
}

[GraphQLObject(Implements = typeof(INode))]
public class Book : INode
{
    public string Id { get; set; }
}
```

### `[GraphQLUnion]`
Defines a GraphQL union type.

```csharp
[GraphQLUnion(typeof(Book), typeof(Magazine), typeof(Newspaper))]
public abstract class SearchResult { }

// Usage in queries
[GraphQLField]
public SearchResult Search(string query) => query.Contains("book") 
    ? new Book() 
    : new Magazine();
```

### `[GraphQLEnum]`
Marks an enum as a GraphQL enum type.

```csharp
[GraphQLEnum]
public enum BookStatus
{
    [GraphQLValue("DRAFT", Description = "Book is being written")]
    Draft,
    
    [GraphQLValue("PUBLISHED")]
    Published,
    
    [GraphQLValue("OUT_OF_PRINT")]
    OutOfPrint
}
```

## Schema Configuration

### `[GraphQLSchema]`
Defines the root schema configuration.

```csharp
[GraphQLSchema]
public class BookStoreSchema
{
    [Query]
    public Query Query { get; set; }
    
    [Mutation]
    public Mutation Mutation { get; set; }
    
    [Subscription]
    public Subscription Subscription { get; set; }
}
```

### Query Root
```csharp
[GraphQLObject]
public class Query
{
    [GraphQLField]
    public Book? GetBook(int id, [Service] IBookService service)
        => service.GetById(id);
    
    [GraphQLField]
    public IQueryable<Book> Books([Service] IBookService service)
        => service.GetAll();
    
    [GraphQLField(Name = "search")]
    public async Task<IEnumerable<SearchResult>> SearchBooks(
        string query, 
        [Service] ISearchService search)
        => await search.SearchAsync(query);
}
```

### Mutation Root
```csharp
[GraphQLObject]
public class Mutation
{
    [GraphQLField]
    public async Task<Book> CreateBook(
        CreateBookInput input,
        [Service] IBookService service)
    {
        return await service.CreateAsync(input);
    }
    
    [GraphQLField]
    [GraphQLTransaction] // Ensures transactional behavior
    public async Task<BatchResult> CreateBooks(
        CreateBookInput[] inputs,
        [Service] IBookService service)
    {
        return await service.CreateBatchAsync(inputs);
    }
}
```

## Field Configuration

### `[GraphQLField]`
Configures a field on a GraphQL type.

```csharp
public class Book
{
    // Simple field - automatically exposed
    public string Title { get; set; }
    
    // Field with custom configuration
    [GraphQLField(
        Name = "isbn",
        Description = "International Standard Book Number",
        DeprecationReason = "Use 'identifier' instead")]
    public string ISBN { get; set; }
    
    // Computed field
    [GraphQLField]
    public string DisplayName => $"{Title} ({Year})";
    
    // Async field with service injection
    [GraphQLField]
    public async Task<Author> Author(
        [Service] IAuthorService service)
        => await service.GetByIdAsync(AuthorId);
    
    // Field with arguments
    [GraphQLField]
    public decimal GetPrice(
        [GraphQLArgument] Currency currency = Currency.USD,
        [GraphQLArgument] bool includeVat = false)
    {
        var price = BasePrice;
        if (includeVat) price *= 1.2m;
        return currency == Currency.EUR ? price * 0.85m : price;
    }
}
```

### `[GraphQLIgnore]`
Prevents a property from being exposed in the GraphQL schema.

```csharp
public class User
{
    public string Email { get; set; }
    
    [GraphQLIgnore]
    public string PasswordHash { get; set; } // Never exposed
    
    [GraphQLIgnore]
    internal int InternalId { get; set; }
}
```

### `[GraphQLArgument]`
Configures method parameters as GraphQL arguments.

```csharp
[GraphQLField]
public async Task<PagedResult<Book>> GetBooks(
    [GraphQLArgument(Description = "Number of items to return")] 
    int first = 10,
    
    [GraphQLArgument(Description = "Cursor for pagination")] 
    string? after = null,
    
    [GraphQLArgument] 
    BookFilter? filter = null,
    
    [GraphQLArgument] 
    BookSort sort = BookSort.Title,
    
    [Service] IBookService service)
{
    return await service.GetPagedAsync(first, after, filter, sort);
}
```

## Service Registration

### Basic Registration
```csharp
var builder = WebApplication.CreateBuilder(args);

// Register GraphQL with default options
builder.Services.AddGraphQL<BookStoreSchema>();

// Register with configuration
builder.Services.AddGraphQL<BookStoreSchema>(options =>
{
    options.MaxQueryDepth = 10;
    options.MaxQueryComplexity = 1000;
    options.EnableIntrospection = !builder.Environment.IsProduction();
    options.EnableMetrics = true;
});

var app = builder.Build();

// Map GraphQL endpoint
app.MapGraphQL("/graphql");

// Optional: Map GraphQL playground
if (app.Environment.IsDevelopment())
{
    app.MapGraphQLPlayground("/graphql/playground");
}

app.Run();
```

### Advanced Configuration
```csharp
builder.Services.AddGraphQL<BookStoreSchema>(options =>
{
    // Performance
    options.QueryCacheSize = 100;
    options.EnableQueryPlanCaching = true;
    options.ParallelFieldResolution = true;
    
    // Security
    options.MaxQueryDepth = 15;
    options.MaxQueryComplexity = 5000;
    options.EnableIntrospection = false;
    options.AllowedOrigins = new[] { "https://app.example.com" };
    
    // Execution
    options.DefaultTimeout = TimeSpan.FromSeconds(30);
    options.IncludeExceptionDetails = builder.Environment.IsDevelopment();
    
    // Extensions
    options.UseAuthorization();
    options.UseDataLoader();
    options.UseMetrics();
});
```

## Advanced Features

### DataLoader
```csharp
[GraphQLDataLoader]
public class AuthorDataLoader : DataLoaderBase<int, Author>
{
    private readonly IAuthorService _service;
    
    public AuthorDataLoader(IAuthorService service)
    {
        _service = service;
    }
    
    protected override async Task<IReadOnlyDictionary<int, Author>> LoadBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken cancellationToken)
    {
        var authors = await _service.GetByIdsAsync(keys, cancellationToken);
        return authors.ToDictionary(a => a.Id);
    }
}

// Usage in field
[GraphQLField]
public async Task<Author> Author(
    [DataLoader] AuthorDataLoader loader)
    => await loader.LoadAsync(AuthorId);
```

### Authorization
```csharp
[GraphQLObject]
[Authorize] // Requires authentication for all fields
public class User
{
    public string Id { get; set; }
    
    [GraphQLField]
    public string Name { get; set; }
    
    [GraphQLField]
    [Authorize(Roles = "Admin")] // Additional authorization
    public string Email { get; set; }
    
    [GraphQLField]
    [AllowAnonymous] // Override class-level auth
    public string PublicBio { get; set; }
}
```

### Custom Directives
```csharp
[GraphQLDirective("uppercase")]
public class UppercaseDirective : IFieldDirective
{
    public ValueTask<object?> InvokeAsync(
        IDirectiveContext context,
        FieldDelegate next)
    {
        var result = await next(context);
        return result is string str ? str.ToUpper() : result;
    }
}

// Usage
[GraphQLField]
[GraphQLDirective("uppercase")]
public string Title { get; set; }
```

### Custom Scalars
```csharp
[GraphQLScalar("DateTime")]
public class DateTimeType : ScalarType<DateTime>
{
    public override string Serialize(DateTime value)
        => value.ToString("O"); // ISO 8601
    
    public override DateTime Parse(string value)
        => DateTime.Parse(value, null, DateTimeStyles.RoundtripKind);
    
    public override bool IsValidValue(string value)
        => DateTime.TryParse(value, out _);
}

// Registration
builder.Services.AddGraphQL<BookStoreSchema>(options =>
{
    options.AddScalarType<DateTimeType>();
});
```

### Middleware
```csharp
[GraphQLMiddleware]
public class TimingMiddleware : IFieldMiddleware
{
    private readonly ILogger<TimingMiddleware> _logger;
    
    public TimingMiddleware(ILogger<TimingMiddleware> logger)
    {
        _logger = logger;
    }
    
    public async ValueTask<object?> InvokeAsync(
        IMiddlewareContext context,
        FieldDelegate next)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            return await next(context);
        }
        finally
        {
            if (sw.ElapsedMilliseconds > 100)
            {
                _logger.LogWarning(
                    "Slow field execution: {Field} took {Duration}ms",
                    context.Field.Name,
                    sw.ElapsedMilliseconds);
            }
        }
    }
}
```

## Extension Points

### Custom Type Generation
```csharp
[AttributeUsage(AttributeTargets.Class)]
public class GraphQLCustomTypeAttribute : Attribute
{
    public Type GeneratorType { get; set; }
}

[GraphQLCustomType(GeneratorType = typeof(MyCustomGenerator))]
public class ComplexType { }
```

### Schema Transformation
```csharp
public class SchemaTransformer : ISchemaTransformer
{
    public void Transform(ISchemaBuilder builder)
    {
        // Add custom types
        builder.AddType<CustomScalar>();
        
        // Modify existing types
        builder.ModifyType<Book>(type =>
        {
            type.AddField("customField", () => "custom value");
        });
    }
}

// Registration
builder.Services.AddGraphQL<BookStoreSchema>(options =>
{
    options.AddSchemaTransformer<SchemaTransformer>();
});
```

### Query Complexity Calculation
```csharp
[GraphQLField(Complexity = 10)]
public IQueryable<Book> Books() => _books;

[GraphQLField]
[GraphQLComplexity("pageSize * 2")] // Dynamic complexity
public PagedResult<Book> GetBooks(int pageSize = 10) => ...;

// Custom complexity calculator
public class CustomComplexityCalculator : IComplexityCalculator
{
    public int Calculate(IFieldSelection field)
    {
        // Custom logic
        return field.Arguments.ContainsKey("includeDetails") ? 50 : 10;
    }
}
```

## Error Handling

### Custom Error Formatting
```csharp
builder.Services.AddGraphQL<BookStoreSchema>(options =>
{
    options.FormatError = error =>
    {
        // Custom error formatting
        return new GraphQLError
        {
            Message = error.Message,
            Code = DetermineErrorCode(error),
            Extensions = new Dictionary<string, object?>
            {
                ["timestamp"] = DateTime.UtcNow,
                ["traceId"] = Activity.Current?.TraceId.ToString()
            }
        };
    };
});
```

### Field Error Handling
```csharp
[GraphQLField]
public async Task<Book?> GetBook(int id, [Service] IBookService service)
{
    try
    {
        return await service.GetByIdAsync(id);
    }
    catch (BookNotFoundException)
    {
        // Return null for not found
        return null;
    }
    catch (Exception ex)
    {
        // Add error to GraphQL response
        throw new GraphQLException($"Failed to load book {id}", ex);
    }
}
```