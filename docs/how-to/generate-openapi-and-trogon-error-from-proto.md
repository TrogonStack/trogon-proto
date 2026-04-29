# Generate OpenAPI and trogon_error from Proto

This guide describes the preferred generation strategy for OpenAPI error schemas and Elixir `trogon_error` modules.

## Goal

Use one proto definition as the source of truth for:

- the canonical error schema
- OpenAPI response schemas
- Google RPC compatibility mappings
- generated `trogon_error` adapters

## Recommended Source of Truth

Do not try to infer the full contract from raw `google.rpc.Status`.

Instead:

1. Define the canonical error model in proto following ADR#0129349218.
2. Define reason-specific typed error payload messages.
3. Attach custom options that describe the static error template.
4. Generate transport and language-specific adapters from that proto.

## Why Not Generate from `google.rpc.Status` Alone

`google.rpc.Status` is not rich enough to describe the full contract because:

- `details` uses `Any`
- `ErrorInfo.metadata` is `map<string,string>`
- OpenAPI generators cannot infer reason-specific metadata structure from that shape

## Proto Shape

Use two kinds of proto definitions.

### 1. Canonical error object

This should model the ADR fields directly in a `trogon.error.*` package.

### 2. Reason-specific typed error payload messages

Each stable reason should get a typed error payload message when the runtime data matters to clients or documentation.

Naming rule:

- if the same domain concept may also appear in non-error shapes, use an explicit `*Error` suffix
- avoid `*Metadata` for the public proto type
- keep `metadata` as a transport/runtime concern handled by adapters

Current simplification:

- only scalar string fields should be supported for proto-to-`trogon_error` generation for now

Example:

```proto
import "trogon/error/v1alpha1/options.proto";

message ResourceAvailabilityError {
  string zone = 1;
  string vm_type = 2;
  string attachment = 3;
  string zones_with_capacity = 4;
}
```

## Template Option Pattern

Add a custom message option to hold the static error template.

Example:

```proto
import "trogon/error/v1alpha1/options.proto";
```

Then annotate the error payload message:

```proto
import "trogon/error/v1alpha1/options.proto";

message ResourceAvailabilityError {
  option (trogon.error.v1alpha1.message).template = {
    domain: "compute.googleapis.com"
    reason: "RESOURCE_AVAILABILITY"
    message: "Requested resources are unavailable."
    code: "RESOURCE_EXHAUSTED"
  };

  string zone = 1;
  string vm_type = 2;
  string attachment = 3;
  string zones_with_capacity = 4;
}
```

## Elixir Integration Direction

The preferred ergonomic direction is to avoid duplicating template fields in Elixir and instead derive them from proto.

Desired shape:

```elixir
use Trogon.Error, proto: MyApp.Protos.ResourceAvailabilityError
```

This guide treats that as the desired API direction. It is not presented as an already released `trogon_error` feature.

## Generated OpenAPI

Preferred path:

1. Generate OpenAPI from the canonical ADR-shaped schema.
2. Use explicit response schema annotations when the generator needs help.
3. Avoid treating raw `google.rpc.Status` as the primary public schema for rich errors.

If `grpc-gateway` is used, it can document explicit response schemas with operation-level OpenAPI options. That is useful for compatibility mode, but it is still better to point those responses at a stronger canonical schema than at plain `Status`.

## Generated trogon_error Adapter

From each reason-specific typed error payload message, generate:

1. A way to derive the static template from proto metadata.
2. A `from_proto/1` constructor or equivalent adapter.
3. A metadata flattening step using the proto JSON field names as the metadata keys.

Example desired runtime shape:

```json
{
  "reason": "RESOURCE_AVAILABILITY",
  "domain": "compute.googleapis.com",
  "metadata": {
    "zone": "us-east1-a",
    "vmType": "e2-medium",
    "attachment": "local-ssd=3,nvidia-t4=2",
    "zonesWithCapacity": "us-central1-f,us-central1-c"
  }
}
```

With the example proto above, the adapter would map:

- `zone` -> `zone`
- `vm_type` -> `vmType`
- `attachment` -> `attachment`
- `zones_with_capacity` -> `zonesWithCapacity`

## Current Limitation

For now, the generation story should assume:

- metadata fields are scalar strings
- no repeated metadata fields
- no nested error payload messages
- no proto-side Elixir module naming hints

If a caller needs list-like data today, it should already be flattened into a string field in the proto contract.

## Recommended Pipeline

1. Define canonical ADR-shaped error proto types.
2. Define string-only reason error payload messages for the cases that need generated adapters.
3. Annotate those messages with template options.
4. Generate:
   - OpenAPI schemas
   - Google RPC compatibility helpers
   - Elixir `trogon_error` integrations

This keeps the contract typed in one place and avoids duplicating the same error definition across HTTP docs, gRPC docs, and language-specific code.

## References

- [../reference/error-model.md](../reference/error-model.md)
- [../explanation/adr-error-model-and-google-rpc-compatibility.md](../explanation/adr-error-model-and-google-rpc-compatibility.md)
- [RESEARCH.md](../../RESEARCH.md)
- https://straw-hat-team.github.io/adr/adrs/0129349218/README.html
- https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/customizing_openapi_output/
- https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/using_ref_with_responses/
- https://hexdocs.pm/trogon_error/Trogon.Error.html
- https://hexdocs.pm/trogon_error/Trogon.Error.Metadata.html
- https://hexdocs.pm/trogon_error/Trogon.Error.MetadataValue.html
