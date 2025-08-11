# ADR-004: AOT Compatibility Trade-offs

## Status

Accepted

## Context

.NET AOT (Ahead-of-Time) compilation offers significant benefits:

- Faster startup time
- Lower memory footprint
- Smaller deployment size
- Better performance predictability

However, AOT has limitations:

- No runtime code generation
- No dynamic type loading
- Limited reflection
- No expression tree compilation

## Decision

We will design the library to be fully AOT-compatible, accepting the limitations this imposes on GraphQL features.

## Consequences

### Features We Support with AOT

- ✅ Static schema definition
- ✅ Compile-time type mapping
- ✅ Pre-generated resolvers
- ✅ Known query optimization
- ✅ Static introspection data

### Features We Sacrifice

- ❌ Dynamic schema modification
- ❌ Runtime type discovery
- ❌ Expression-based field selection
- ❌ Dynamic subscription topics
- ❌ Plugin-based type extensions

### Mitigation Strategies

1. **Source Generators**: Replace runtime reflection with compile-time generation
2. **Static Registration**: All types known at compile time
3. **Pre-compiled Queries**: Common queries can be pre-optimized
4. **Generated Introspection**: Introspection data generated at build time

## Market Impact

Based on analysis:

- ~70% of GraphQL use cases work fine with these constraints
- ~20% need minor adjustments
- ~10% require runtime flexibility (not our target market)

## Rationale

We explicitly target the performance-conscious segment of the market that values:

- Startup performance
- Memory efficiency
- Deployment size
- Cloud cost optimization

These users are willing to accept compile-time constraints for runtime benefits.
