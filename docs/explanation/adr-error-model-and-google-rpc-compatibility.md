# ADR Error Model and Google RPC Compatibility

This document explains the recommended relationship between the Straw Hat error specification, Google RPC errors, and generated API artifacts in trogon-proto.

## Short Answer

The canonical error schema should follow ADR#0129349218. Google RPC should be treated as a compatibility transport for gRPC, not as the source of truth for the repository-wide error contract.

## Why This Matters

The ADR is explicit that the model is:

- data-oriented
- transport-independent
- not tied to a specific wire format

That makes the ADR a better foundation for proto modeling than `google.rpc.Status`, which is already a transport-oriented compatibility shape.

## What the ADR Models Directly

The ADR models all of these as first-class fields:

- `specversion`
- `code`
- `message`
- `domain`
- `reason`
- `metadata`
- `causes`
- `visibility`
- optional support fields such as `subject`, `id`, `time`, `help`, `debug_info`, `localized_message`, `retry_info`, and `source_id`

It also defines semantics that matter independently of transport:

- top-level visibility filtering
- metadata entry visibility filtering
- recursive error composition
- field-level validation via `subject`
- operational help links

## What Google RPC Models Well

Google RPC maps cleanly to several ADR concepts:

- `code`
- `domain`
- `reason`
- plain string metadata
- `Help`
- `DebugInfo`
- `LocalizedMessage`
- `RetryInfo`

That makes it a useful compatibility layer for gRPC.

## What Google RPC Does Not Model Directly

Google RPC does not natively capture several ADR semantics:

- metadata entries as `{value, visibility}`
- top-level error visibility
- recursive `causes` as structured domain errors
- two-tier filtering behavior across trust boundaries

Those gaps are the main reason the canonical repo-level schema should not be just raw `google.rpc.Status`.

## Recommended Architecture

Use this layering:

1. Canonical proto model
   - Define a `trogon.error.*` package that follows the ADR.
2. gRPC compatibility mapping
   - Map canonical errors to `google.rpc.Status` plus compatible details.
3. OpenAPI generation
   - Generate OpenAPI from the canonical proto schema.
4. Language-specific adapters
   - Generate runtime adapters such as `trogon_error` modules.

## Mapping Strategy

Suggested field mapping:

| ADR field | Google RPC compatibility |
|----------|---------------------------|
| `code` | `google.rpc.Code` / `Status.code` |
| `message` | `Status.message` |
| `domain` | `ErrorInfo.domain` |
| `reason` | `ErrorInfo.reason` |
| `metadata` | `ErrorInfo.metadata` after flattening |
| `help` | `google.rpc.Help` |
| `debug_info` | `google.rpc.DebugInfo` |
| `localized_message` | `google.rpc.LocalizedMessage` |
| `retry_info` | `google.rpc.RetryInfo` |

Fields that require extra handling:

| ADR field | Compatibility note |
|----------|---------------------|
| `visibility` | filter before emitting public payloads |
| metadata entry visibility | filter before flattening into `ErrorInfo.metadata` |
| `causes` | may require custom detail payloads or a canonical envelope |
| `id`, `time`, `source_id`, `subject` | may need a custom detail message if preserved over gRPC |

## OpenAPI Implication

If OpenAPI is generated from raw `google.rpc.Status`, the resulting schema will usually be too weak because:

- details are `Any`
- metadata is an untyped string map

If OpenAPI is generated from the canonical ADR-shaped proto, the schema can describe:

- visibility enums
- metadata entry structure
- nested causes
- help links
- retry info
- optional instance fields

## trogon_error Implication

`trogon_error` aligns better with the ADR than with raw Google RPC because it already assumes:

- static template-level `domain`, `reason`, `code`, and `message`
- runtime metadata payloads
- visibility-aware metadata entries

The preferred Elixir-facing direction is to let a module derive its template from proto metadata rather than restating the template manually.

Desired ergonomic shape:

```elixir
use Trogon.Error, proto: MyApp.Protos.ResourceAvailabilityError
```

This document describes that as a target integration style, not as a capability that already exists today.

## Current Simplification

For proto-to-`trogon_error` generation, only scalar string fields should be supported for reason error payload messages for now.

That means values such as:

- `zone`
- `vmType`
- `attachment`
- `zonesWithCapacity`

should all already be represented as strings in the proto error payload message if they are meant to flow directly into `trogon_error` metadata.

## Practical Rule

When deciding where a concept belongs:

- if it is part of the logical error contract, model it in the canonical ADR proto
- if it is specific to gRPC transport compatibility, map it into Google RPC details
- if it is needed for HTTP schema generation, derive it from the canonical proto

## References

- [../reference/error-model.md](../reference/error-model.md)
- [../how-to/generate-openapi-and-trogon-error-from-proto.md](../how-to/generate-openapi-and-trogon-error-from-proto.md)
- [../reference/error-sources.md](../reference/error-sources.md)
- https://straw-hat-team.github.io/adr/adrs/0129349218/README.html
- https://google.aip.dev/193
- https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto
