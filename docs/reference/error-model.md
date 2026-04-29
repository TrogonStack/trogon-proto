# Error Model

This document defines the repository-level error model guidance for trogon-proto.

It is intentionally normative. `RESEARCH.md` records the research trail. This document defines the contract direction the repo should follow.

## Overview

The preferred architecture has two layers:

1. A canonical, transport-independent error schema shaped by Straw Hat ADR#0129349218.
2. A Google RPC compatibility mapping for gRPC services that need `google.rpc.Status`.

The canonical schema should be the source of truth for:

- repo-level documentation
- OpenAPI generation
- language-specific adapters

The Google RPC shape should be treated as a compatibility transport, not as the canonical schema.

## Canonical Model

The canonical error model should preserve the ADR concepts directly:

- `specversion`
- `code`
- `message`
- `domain`
- `reason`
- `metadata`
- `causes`
- `visibility`
- optional fields such as `subject`, `id`, `time`, `help`, `debug_info`, `localized_message`, `retry_info`, and `source_id`

The stable logical identity of an error is:

- `code`
- `domain`
- `reason`

The canonical model should also preserve:

- top-level visibility
- per-metadata-entry visibility
- recursive causes
- trust-boundary filtering semantics

## Google RPC Compatibility

When a service exposes gRPC errors using the Google model, the compatibility shape should use:

- `google.rpc.Status`
- canonical `google.rpc.Code`
- `google.rpc.ErrorInfo`
- standard detail payloads such as `Help`, `DebugInfo`, `LocalizedMessage`, `RetryInfo`, `BadRequest`, `PreconditionFailure`, and `ResourceInfo`

This compatibility layer maps well to:

- `code`
- `domain`
- `reason`
- plain string metadata
- several optional structured details

It does not directly model:

- metadata entries as `{value, visibility}`
- top-level error visibility
- recursive `causes` as first-class domain errors
- the ADR filtering behavior across trust boundaries

For that reason, raw `google.rpc.Status` should not be treated as the strongest source of truth for the repo.

## Required Shape

Canonical repo error models should:

1. Preserve the ADR field set and semantics.
2. Use stable `code`, `domain`, and `reason` values.
3. Keep metadata keys stable after publication.
4. Support nested `causes`.
5. Preserve visibility semantics before transport-specific filtering.

Google RPC compatibility layers should:

1. Return `google.rpc.Status` for API errors.
2. Use canonical `google.rpc.Code` values.
3. Include `google.rpc.ErrorInfo` in `Status.details`.
4. Use a stable `(reason, domain)` pair for the same logical error.
5. Keep `ErrorInfo.metadata` keys stable once introduced for a `(reason, domain)` pair.

## Status Codes

The top-level status code answers:

- what broad class of error occurred
- whether the problem is usually client-side, state-related, or transient

Examples:

- `INVALID_ARGUMENT`: the request is malformed regardless of current system state
- `NOT_FOUND`: the requested resource does not exist
- `ALREADY_EXISTS`: creation conflicts with an existing resource
- `FAILED_PRECONDITION`: the request is valid, but the system is not in the required state
- `PERMISSION_DENIED`: the caller is not authorized
- `UNAVAILABLE`: the system is temporarily unable to serve the request

The status code is necessary but not sufficient for a stable contract. Distinct failures may share the same code.

## Domain and Reason

Whether the error is represented in the canonical ADR shape or the Google RPC compatibility shape, `domain` and `reason` remain the stable logical identifier for client handling.

### Reason

`reason` should:

- be stable
- be unique within a `domain`
- use `UPPER_SNAKE_CASE`
- identify a specific client-handling case

Good examples:

- `WIDGET_NOT_FOUND`
- `WIDGET_HAS_CHILDREN`
- `RESOURCE_AVAILABILITY`
- `PROJECTION_TIMEOUT`

Weak examples:

- `ERROR`
- `BAD_REQUEST`
- `DELETE_FAILED`

### Domain

`domain` should be globally unique and usually match the service that owns the error contract.

Good examples:

- `compute.googleapis.com`
- `widgets.trogonstack.com`
- `orders.trogonstack.com`

The same `reason` string may appear in different domains, but the `(reason, domain)` pair must remain unique in meaning.

## Metadata

In the canonical ADR shape, metadata entries carry both:

- a string value
- a visibility level

In the Google RPC compatibility shape, metadata becomes:

- `ErrorInfo.metadata`

That flattening step is one reason the canonical proto should remain the source of truth.

Metadata keys should:

- use lowerCamelCase
- remain stable once introduced
- be documented as required or optional
- include units in the key name rather than the value when units matter

Good examples:

- `zone`
- `vmType`
- `attachment`
- `zonesWithCapacity`
- `requestedVersion`

## Documentation Contract

This repo documents errors at three layers.

### 1. Canonical Error Schema

The canonical proto schema should define:

- the error object shape
- visibility semantics
- nested causes
- typed companion error payload messages when useful

### 2. Shared Reason Catalog

Each service or domain should maintain a shared reason catalog.

That catalog should define:

- the stable reason values
- the owning domain
- the default or typical canonical code
- the supported metadata keys
- sample payloads

See [error-reasons.md](./error-reasons.md).

### 3. RPC Comments

When an API uses Google-style gRPC errors, each RPC comment should document:

- the canonical `google.rpc.Code` values it may return
- the condition that triggers each code
- the applicable `ErrorInfo.reason` values when useful

Example:

```proto
// Deletes a widget.
//
// Returns the following error codes:
// - `NOT_FOUND` with `ErrorInfo.reason = WIDGET_NOT_FOUND` if the widget does not exist.
// - `FAILED_PRECONDITION` with `ErrorInfo.reason = WIDGET_HAS_CHILDREN` if the widget still has child resources.
rpc DeleteWidget(DeleteWidgetRequest) returns (google.protobuf.Empty);
```

## Standard Detail Payloads

Do not overuse `ErrorInfo.metadata` for information that already has a better standard detail message.

Prefer:

- `BadRequest` for request field violations
- `PreconditionFailure` for unmet invariants
- `ResourceInfo` for resource identity and ownership context
- `RetryInfo` for retry timing hints
- `Help` for troubleshooting links
- `LocalizedMessage` for user-facing text

These detail messages belong to the Google RPC compatibility layer. They do not replace the canonical ADR-shaped model.

## OpenAPI and Code Generation

If this repo wants precise OpenAPI error schemas, they should be generated from the canonical ADR-shaped proto, not inferred from raw `google.rpc.Status`.

Why:

- `Status.details` uses `Any`
- `ErrorInfo.metadata` is `map<string,string>`
- neither captures the ADR visibility model directly

The preferred pipeline is:

1. Define the canonical error proto.
2. Generate OpenAPI from that proto.
3. Generate compatibility mappings to `google.rpc.Status` for gRPC.
4. Generate language-specific adapters such as `trogon_error`.

See:

- [../explanation/adr-error-model-and-google-rpc-compatibility.md](../explanation/adr-error-model-and-google-rpc-compatibility.md)
- [../how-to/generate-openapi-and-trogon-error-from-proto.md](../how-to/generate-openapi-and-trogon-error-from-proto.md)

## What This Repo Does Not Assume

This repo does not assume that:

- Google AIP provides a standard machine-readable per-RPC error annotation in proto.
- raw `google.rpc.Status` is rich enough to serve as the canonical repo-level error schema.
- a generated Elixir adapter should require proto-side module naming hints.

If a project wants machine-readable per-method contracts, that should be modeled with custom options owned by the project.

## References

- [RESEARCH.md](../../RESEARCH.md)
- https://straw-hat-team.github.io/adr/adrs/0129349218/README.html
- https://google.aip.dev/192
- https://google.aip.dev/193
- https://google.aip.dev/194
- https://google.aip.dev/client-libraries/4221
- https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
- https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto
- https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo
- https://cloud.google.com/storage/docs/reference/rpc/google.rpc
- https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/customizing_openapi_output/
- https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/using_ref_with_responses/
- https://hexdocs.pm/trogon_error/Trogon.Error.html
- https://hexdocs.pm/trogon_error/Trogon.Error.Metadata.html
- https://hexdocs.pm/trogon_error/Trogon.Error.MetadataValue.html
- https://grpc.io/docs/guides/status-codes/
