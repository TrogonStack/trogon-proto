# trogon-proto

**Shared Protocol Buffer definitions and custom options for TrogonStack projects.**

**trogon-proto provides reusable protobuf extensions that standardize common patterns across services.** It includes custom options for UUID generation, identity versioning, and other cross-cutting concerns.

**Protocol Buffer options let you attach metadata to your definitions that can be read at runtime or during code generation.** This eliminates documentation drift and ensures all services follow the same conventions for critical patterns like deterministic ID generation.

**trogon-proto is designed for teams building event-sourced systems or microservices where consistent identity generation matters.** It integrates with Buf and standard protoc toolchains.

## Installation

Add to your `buf.yaml`:

```yaml
deps:
  - buf.build/trogonstack/trogon-proto
```

## Excluding Elixir-Specific Files

The `trogon-proto` module includes `elixirpb.proto`, which provides Elixir-specific protobuf extensions. To safely exclude this file when generating code, use the `paths` option in your `buf.gen.yaml`:

```yaml
managed:
  - module: buf.build/trogonstack/trogon-proto:v0.2.1
    paths:
      - trogon
```

This generates code only from the `trogon` directory, which contains all shared protobuf extensions like UUID options and identity versioning. The `paths` option is safe for all consumers regardless of language.

**Note:** We had to commit `elixirpb.proto` to this repository because the Buf Schema Registry doesn't currently support distributing external dependencies like the standard Elixir protobuf definitions. Once BSR support improves, we can remove this file.

