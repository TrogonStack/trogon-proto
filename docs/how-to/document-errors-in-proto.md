# Document Errors in Proto

This guide shows the intended protobuf shape for typed error payload messages that use `trogon.error.v1alpha1`.

## Example

```proto
import "trogon/error/v1alpha1/code.proto";
import "trogon/error/v1alpha1/options.proto";
import "trogon/error/v1alpha1/visibility.proto";

message ResourceAvailabilityError {
  option (trogon.error.v1alpha1.message).template = {
    domain: "compute.googleapis.com"
    reason: "RESOURCE_AVAILABILITY"
    message: "Requested resources are unavailable."
    code: RESOURCE_EXHAUSTED
    visibility: VISIBILITY_PUBLIC
    help_links: [
      {url: "https://docs.acme.com/compute", description: "Compute Docs"}
    ]
    metadata: [
      {key: "component", value: "compute", visibility: VISIBILITY_PUBLIC},
      {key: "team", value: "platform-compute", visibility: VISIBILITY_INTERNAL}
    ]
  };

  string zone = 1;
  string vm_type = 2;
  string service = 3 [(trogon.error.v1alpha1.field) = {
    visibility: VISIBILITY_PUBLIC,
    default_value: "compute-api"
  }];
  string region = 4 [(trogon.error.v1alpha1.field) = {
    visibility: VISIBILITY_PUBLIC,
    value: "us-east-1"
  }];
}
```

## Notes

- `template` is a message field, so it carries proto3 presence by default.
- The template captures the parts of the error contract that never vary at runtime: `domain`, `reason`, `code`, default `message`, error-level `visibility`, `help_links`, and fixed `metadata` entries.
- `metadata` declares contract-level constants (component, team, subsystem) that attach to every emission without occupying a wire field. Keys must be unique within the template and must not collide with field names on the payload message.
- Payload fields hold the dynamic context for a single emission. Each field can carry `FieldOptions`:
  - `visibility` controls who sees the field at runtime (defaults to `INTERNAL`).
  - `value_policy.default_value` substitutes a value when the runtime field is empty.
  - `value_policy.value` pins the field to a contract-fixed value; the runtime payload is ignored on emit.

## References

- [../../proto/trogon/error/v1alpha1/code.proto](../../proto/trogon/error/v1alpha1/code.proto)
- [../../proto/trogon/error/v1alpha1/options.proto](../../proto/trogon/error/v1alpha1/options.proto)
- [../../proto/trogon/error/v1alpha1/visibility.proto](../../proto/trogon/error/v1alpha1/visibility.proto)
