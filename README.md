# trogon-proto

**Shared Protocol Buffer definitions and custom options for TrogonStack projects.**

**trogon-proto provides reusable protobuf extensions that standardize common patterns across services.** It includes custom options for UUID generation, identity versioning, and other cross-cutting concerns.

**Protocol Buffer options let you attach metadata to your definitions that can be read at runtime or during code generation.** This eliminates documentation drift and ensures all services follow the same conventions for critical patterns like deterministic ID generation.

**trogon-proto is designed for teams building event-sourced systems or microservices where consistent identity generation matters.** It integrates with Buf and standard protoc toolchains.

## Documentation

- [Error Model](docs/reference/error-model.md)
- [Error Reasons](docs/reference/error-reasons.md)
- [Document Errors in Proto](docs/how-to/document-errors-in-proto.md)
- [Generate OpenAPI and trogon_error from Proto](docs/how-to/generate-openapi-and-trogon-error-from-proto.md)
- [ADR Error Model and Google RPC Compatibility](docs/explanation/adr-error-model-and-google-rpc-compatibility.md)
- [Error Detail Selection](docs/explanation/error-detail-selection.md)
- [Research: Documenting Google-Style gRPC Errors in Proto](RESEARCH.md)

## Installation

Add to your `buf.yaml`:

```yaml
deps:
  - buf.build/trogonstack/trogon-proto
```
