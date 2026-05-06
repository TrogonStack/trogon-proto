# Document Errors in Proto

This guide shows the intended protobuf shape for typed error payload messages that use `trogon.error.v1alpha1`.

Trogon error annotations are descriptor-time templates for the Google RPC rich error model. They describe the `google.rpc.Status` and `google.rpc.ErrorInfo` details a runtime should emit when the typed error occurs.

This is the protobuf projection of [ADR 0129349218: Error Specification](https://straw-hat-team.github.io/adr/adrs/0129349218/README.html). The ADR explains the full boundary model; this guide only documents the descriptor fields that are safe to share.

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
      {key: "component", value: "compute", visibility: VISIBILITY_PUBLIC}
    ]
  };

  string zone = 1 [(trogon.error.v1alpha1.field).visibility = VISIBILITY_PUBLIC];
  string vm_type = 2 [(trogon.error.v1alpha1.field).visibility = VISIBILITY_PUBLIC];
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

## Runtime Mapping

When a runtime emits this error, it should build the protocol-native error envelope from the template:

| Trogon option | Google RPC field |
|---------------|------------------|
| `code` | `google.rpc.Status.code` |
| `message` | `google.rpc.Status.message` |
| `domain` | `google.rpc.ErrorInfo.domain` |
| `reason` | `google.rpc.ErrorInfo.reason` |
| `metadata` | `google.rpc.ErrorInfo.metadata` |
| Payload fields | `google.rpc.ErrorInfo.metadata` using proto JSON names |
| `help_links` | `google.rpc.Help.links` |

For the example above, a runtime could emit this JSON representation of a `google.rpc.Status`:

```json
{
  "code": 8,
  "message": "Requested resources are unavailable.",
  "details": [
    {
      "@type": "type.googleapis.com/google.rpc.ErrorInfo",
      "reason": "RESOURCE_AVAILABILITY",
      "domain": "compute.googleapis.com",
      "metadata": {
        "component": "compute",
        "region": "us-east-1",
        "service": "compute-api",
        "vmType": "n2-standard-16",
        "zone": "us-east1-b"
      }
    },
    {
      "@type": "type.googleapis.com/google.rpc.Help",
      "links": [
        {
          "description": "Compute Docs",
          "url": "https://docs.acme.com/compute"
        }
      ]
    }
  ]
}
```

## Notes

- `template` is a message field, so it carries proto3 presence by default.
- The template captures the parts of the Google RPC error contract that never vary at runtime: `domain`, `reason`, `code`, default `message`, error-level `visibility`, `help_links`, and fixed `metadata` entries.
- `domain` and `reason` describe the `google.rpc.ErrorInfo` identity for this error. `reason` must be stable within the `domain`.
- `metadata` declares contract-level constants (component, product, support category) that attach to every emission without occupying a wire field. Keys should use lowerCamelCase, must be unique within the template, and must not collide with field names on the payload message.
- Payload fields hold the dynamic context for a single emission. Each field can carry `FieldOptions`:
  - `visibility` controls who sees the field at runtime. `VISIBILITY_UNSPECIFIED` is invalid for emitted metadata.
  - `value_policy.default_value` substitutes a value when the runtime field is empty.
  - `value_policy.value` pins the field to a contract-fixed value; the runtime payload is ignored on emit.
- Payload fields use their protobuf JSON names as `ErrorInfo.metadata` keys. For example, `vm_type` emits `vmType`.
- Shared protos should use only `VISIBILITY_PUBLIC` and `VISIBILITY_PRIVATE`. Internal-only metadata belongs in runtime enrichment or internal overlays, not shared descriptors.
- `visibility` is a Trogon filtering policy. It is not part of `google.rpc.ErrorInfo`.

## References

- [ADR 0129349218: Error Specification](https://straw-hat-team.github.io/adr/adrs/0129349218/README.html)
- [../../proto/trogon/error/v1alpha1/code.proto](../../proto/trogon/error/v1alpha1/code.proto)
- [../../proto/trogon/error/v1alpha1/options.proto](../../proto/trogon/error/v1alpha1/options.proto)
- [../../proto/trogon/error/v1alpha1/visibility.proto](../../proto/trogon/error/v1alpha1/visibility.proto)
- [Google RPC Status](https://cloud.google.com/tasks/docs/reference/rpc/google.rpc#status)
- [Google RPC ErrorInfo](https://cloud.google.com/spanner/docs/reference/rpc/google.rpc#errorinfo)
