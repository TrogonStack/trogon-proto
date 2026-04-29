# Document Errors in Proto

This guide documents the protobuf shape used by `proto/trogon/error/v1alpha1/options.proto`.

Use it when defining a typed error payload message that should carry a static error template.

## Import

```proto
import "trogon/error/v1alpha1/options.proto";
```

## Message Naming

Use `*Error` for public typed error payload messages when the same concept may also appear in non-error message types.

Prefer:

- `ResourceAvailabilityError`
- `ProjectionTimeoutError`

Avoid:

- `ResourceAvailabilityMetadata`
- `ProjectionTimeoutMetadata`

## Template Option

Annotate the message with `(trogon.error.v1alpha1.message).template`.

```proto
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

## Template Fields

- `domain`: logical owner of the error contract
- `reason`: stable machine-readable reason name
- `message`: default human-readable message template
- `code`: canonical code name as a string

`code` is currently a string field in the option itself.

## Field Shape

The message fields carry the dynamic payload associated with the error template.

Current limitation:

- only scalar `string` fields are supported for this pattern

Do not use repeated or nested payload fields for this option surface yet.

## Metadata Key Names

When a runtime adapter flattens the payload into string metadata, use the proto JSON field name.

Examples:

- `zone` -> `zone`
- `vm_type` -> `vmType`
- `attachment` -> `attachment`
- `zones_with_capacity` -> `zonesWithCapacity`

## Checklist

- The message name ends with `Error` when it needs to stay distinct from non-error shapes.
- The message imports `trogon/error/v1alpha1/options.proto`.
- The message sets `(trogon.error.v1alpha1.message).template`.
- `domain`, `reason`, `message`, and `code` are all present.
- Dynamic payload fields are scalar `string` fields.

## References

- [../reference/error-sources.md](../reference/error-sources.md)
- [../../proto/trogon/error/v1alpha1/options.proto](../../proto/trogon/error/v1alpha1/options.proto)
