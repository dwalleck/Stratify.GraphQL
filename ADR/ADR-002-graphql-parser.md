# ADR-002: Leverage GraphQLParser Instead of Custom Parser

## Status

Accepted

## Context

We need to parse GraphQL queries at runtime. We have two options:

1. Build our own GraphQL parser
2. Use the existing GraphQLParser library

GraphQLParser is:

- Battle-tested (used by HotChocolate, GraphQL for .NET)
- Spec-compliant
- Well-maintained
- Zero dependencies
- Actively developed

## Decision

We will use GraphQLParser for all query parsing needs rather than building our own parser.

## Consequences

### Positive

- **Time to Market**: Focus on our core value proposition (execution optimization)
- **Correctness**: Handles all GraphQL edge cases correctly
- **Maintenance**: Benefit from community updates and fixes
- **Performance**: Already optimized, parsing is typically <5% of request time
- **Compatibility**: Same parsing behavior as other major GraphQL libraries

### Negative

- **External Dependency**: Relying on third-party library
- **Less Control**: Cannot optimize parsing for our specific use cases
- **Size**: Adds to overall package size

### Neutral

- **Customization**: May need wrapper if we want custom extensions later
- **Abstraction**: Should abstract behind interface for future flexibility

## Rationale

Parsing is a commodity feature that doesn't differentiate our library. Our value comes from:

- Source generation
- Execution optimization
- Developer experience
- AOT compatibility

Building a parser would be significant effort for minimal benefit. GraphQLParser is mature, reliable, and allows us to focus on our unique value propositions.
