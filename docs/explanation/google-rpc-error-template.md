# Google RPC Error Templates

Trogon error options describe the Google RPC rich error details a runtime should emit. They are descriptor metadata, not a replacement for gRPC, Connect, or HTTP error envelopes.

## Why This Exists

This package is based on [ADR 0129349218: Error Specification](https://straw-hat-team.github.io/adr/adrs/0129349218/README.html). The ADR defines the broader error model: stable error identity, structured metadata, visibility-aware filtering, help links, retry guidance, and boundary processing.

`trogon.error.v1alpha1` is the shared-protobuf projection of that model. It keeps the pieces that can safely live in published descriptors and maps them to Google RPC rich error details.

Without a shared proto annotation, every runtime has to construct `google.rpc.Status`, `google.rpc.ErrorInfo`, and related details by hand. That makes stable fields such as `domain`, `reason`, and metadata keys easy to drift across languages.

The `trogon.error.v1alpha1` package keeps that contract close to the typed error payload message. Code generators and runtime adapters can read the same descriptor metadata and produce consistent Google RPC details.

## Mapping

| Trogon template field | Google RPC target |
|-----------------------|-------------------|
| `code` | `google.rpc.Status.code` |
| `message` | `google.rpc.Status.message` |
| `domain` | `google.rpc.ErrorInfo.domain` |
| `reason` | `google.rpc.ErrorInfo.reason` |
| `metadata` | `google.rpc.ErrorInfo.metadata` |
| `help_links` | `google.rpc.Help.links` |

Payload fields annotated with `trogon.error.v1alpha1.field` supply emission-specific `ErrorInfo.metadata` values. Template metadata supplies fixed metadata values that apply to every emission. Metadata keys should use lowerCamelCase; payload fields use their protobuf JSON names.

## Boundaries

Trogon error templates do not declare which RPC can return an error. That belongs in a method-level outcome option.

Trogon error templates also do not model successful outcomes. `trogon.error.v1alpha1.Code` intentionally mirrors only Google RPC error codes and omits `OK`.

Shared error protos should expose only public or private metadata. Internal-only metadata belongs in runtime enrichment, observability pipelines, or internal-only overlays because descriptor annotations are visible to anyone who receives the proto.

This is the intentional difference from the full ADR model. The ADR still describes internal runtime errors and internal metadata filtering. Shared protos do not name that visibility because naming an internal descriptor value invites accidental publication.

The runtime owns protocol adaptation. A gRPC runtime can emit `google.rpc.Status` with typed details. A Connect runtime can expose the same semantics through Connect errors and strongly typed details.

## Example

A template like this:

```proto
message UserNotFoundError {
  option (trogon.error.v1alpha1.message).template = {
    domain: "identity.trogonstack.dev"
    reason: "USER_NOT_FOUND"
    message: "The requested user was not found."
    code: NOT_FOUND
    visibility: VISIBILITY_PUBLIC
  };

  string user_id = 1 [(trogon.error.v1alpha1.field).visibility = VISIBILITY_PUBLIC];
}
```

describes this Google RPC detail identity:

```json
{
  "@type": "type.googleapis.com/google.rpc.ErrorInfo",
  "domain": "identity.trogonstack.dev",
  "reason": "USER_NOT_FOUND",
  "metadata": {
    "userId": "usr_123"
  }
}
```

The surrounding transport error still comes from the runtime protocol. Trogon only defines the stable error detail contract.

## References

- [ADR 0129349218: Error Specification](https://straw-hat-team.github.io/adr/adrs/0129349218/README.html)
- [Document Errors in Proto](../how-to/document-errors-in-proto.md)
- [Google RPC Status](https://cloud.google.com/tasks/docs/reference/rpc/google.rpc#status)
- [Google RPC ErrorInfo](https://cloud.google.com/spanner/docs/reference/rpc/google.rpc#errorinfo)
