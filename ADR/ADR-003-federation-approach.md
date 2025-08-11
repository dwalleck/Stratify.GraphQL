# ADR-003: Build-Time Federation Over Runtime Composition

## Status

Accepted

## Context

GraphQL Federation allows composing multiple GraphQL services into a unified schema. Traditional approaches use runtime service discovery and dynamic schema composition.

For our microservices architecture where service topology is known at build time, we need to decide between:

1. Runtime federation (discover services dynamically)
2. Build-time federation (compose services at compile time)

## Decision

We will implement build-time federation where service schemas are composed during compilation, generating static query planners and service clients.

## Consequences

### Positive

- **Performance**: No runtime service discovery overhead
- **Type Safety**: Service contracts validated at compile time
- **Predictability**: Query plans are pre-computed and optimized
- **Debugging**: Generated federation code is inspectable
- **AOT Compatible**: No dynamic proxy generation

### Negative

- **Less Flexible**: Cannot add/remove services without recompilation
- **Deployment Coupling**: Gateway must be redeployed when service schemas change
- **Limited Scenarios**: Not suitable for dynamic service discovery

### Neutral

- **Different Workflow**: Requires schema files available at build time
- **Versioning**: Need strategy for schema evolution

## Implementation

```csharp
[FederatedSchema]
public partial class Gateway
{
    [FederatedService("UserService", SchemaFile = "../UserService/schema.graphql")]
    [FederatedService("ProductService", SchemaFile = "../ProductService/schema.graphql")]
    public void ConfigureServices() { }
}
```

## Trade-offs

We optimize for:

- Known service topology (common in enterprise microservices)
- Maximum performance
- Compile-time safety

At the cost of:

- Dynamic service discovery
- Runtime flexibility
