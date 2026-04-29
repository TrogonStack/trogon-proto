# Error Detail Selection

This document explains how to choose between `google.rpc.ErrorInfo` and the richer standard detail payloads in `google.rpc.Status.details`.

This guidance applies to the Google RPC compatibility layer. It does not replace the canonical ADR-shaped error model described in [../reference/error-model.md](../reference/error-model.md).

## Why This Matters

`ErrorInfo` is required for the stable machine-readable identity of an error, but it is not the only detail type available.

If every error is flattened into `ErrorInfo.metadata`, clients lose structure and authors end up re-creating existing detail messages by hand.

The right question is not:

- should we use `ErrorInfo` or other details

The right question is:

- which other details should accompany `ErrorInfo`

## Default Rule

Use `ErrorInfo` for:

- stable machine-readable identity
- service/domain ownership
- lightweight dynamic context

Use additional detail messages for richer structure.

## Selection Guide

| Detail type | Use it for | Avoid using it for |
|------------|------------|--------------------|
| `ErrorInfo` | stable reason, domain, lightweight metadata | field-level validation structure |
| `BadRequest` | invalid request fields and validation errors | business preconditions unrelated to request shape |
| `PreconditionFailure` | unmet invariants or state requirements | malformed input syntax |
| `ResourceInfo` | resource type, name, owner, access target | general retry or validation information |
| `RetryInfo` | retry delay hints | describing why a business rule failed |
| `Help` | troubleshooting links | primary problem definition |
| `LocalizedMessage` | user-facing localized text | machine-readable classification |

## Use `ErrorInfo`

Always use `ErrorInfo` when following the Google AIP error model.

It answers:

- what stable error happened
- who owns the contract
- what dynamic values matter for automation

Example:

```json
{
  "reason": "WIDGET_NOT_FOUND",
  "domain": "widgets.trogonstack.com",
  "metadata": {
    "resourceName": "widgets/123",
    "resourceType": "Widget"
  }
}
```

## Use `BadRequest`

Use `BadRequest` when the request itself is malformed or violates field-level validation rules.

Good cases:

- invalid field format
- missing required field
- invalid enum combination
- malformed filter expression

Pair it with:

- `Code = INVALID_ARGUMENT`
- `ErrorInfo.reason` such as `INVALID_WIDGET_SPEC`

## Use `PreconditionFailure`

Use `PreconditionFailure` when the request is structurally valid but the system state does not allow the operation.

Good cases:

- deleting a resource that still has dependents
- performing an action before setup is complete
- requiring a terms-of-service acceptance
- exact-version consistency that has already moved past the requested version

Pair it with:

- `Code = FAILED_PRECONDITION`
- a specific `ErrorInfo.reason`

## Use `ResourceInfo`

Use `ResourceInfo` when the client benefits from a typed description of the affected resource.

Good cases:

- missing resource
- permission denied on a named resource
- precondition tied to one concrete resource

It often complements:

- `NOT_FOUND`
- `PERMISSION_DENIED`
- `FAILED_PRECONDITION`

## Use `RetryInfo`

Use `RetryInfo` when the error is transient and the server can provide retry timing guidance.

Good cases:

- temporary unavailability
- projection lag timeout
- capacity or throttling behavior with a known retry delay

Do not use it for permanent business-rule failures.

## Use `Help`

Use `Help` when a human may need a troubleshooting or remediation link.

Good cases:

- service enablement flows
- quota troubleshooting
- operational runbooks

`Help` supplements the error. It should not replace a clear code and reason.

## Use `LocalizedMessage`

Use `LocalizedMessage` when user-facing text matters or when `Status.message` must remain stable but a better message is needed for display.

`LocalizedMessage` is for presentation. `ErrorInfo` remains the machine-readable contract.

## Composed Examples

### Invalid request field

- `Code = INVALID_ARGUMENT`
- `ErrorInfo.reason = INVALID_WIDGET_SPEC`
- `BadRequest`

### Missing resource

- `Code = NOT_FOUND`
- `ErrorInfo.reason = WIDGET_NOT_FOUND`
- `ResourceInfo`

### Resource has children

- `Code = FAILED_PRECONDITION`
- `ErrorInfo.reason = WIDGET_HAS_CHILDREN`
- `PreconditionFailure`
- `ResourceInfo`

### Temporary read-model lag

- `Code = UNAVAILABLE`
- `ErrorInfo.reason = PROJECTION_TIMEOUT`
- `RetryInfo`

## Example: Do Not Flatten Everything into Metadata

Weak pattern:

```json
{
  "reason": "INVALID_WIDGET_SPEC",
  "domain": "widgets.trogonstack.com",
  "metadata": {
    "field": "filter",
    "description": "unsupported operator",
    "expectedFormat": "field OP value"
  }
}
```

Better pattern:

```json
{
  "code": 3,
  "message": "Filter expression is invalid.",
  "details": [
    {
      "@type": "type.googleapis.com/google.rpc.ErrorInfo",
      "reason": "INVALID_WIDGET_SPEC",
      "domain": "widgets.trogonstack.com",
      "metadata": {
        "field": "filter"
      }
    },
    {
      "@type": "type.googleapis.com/google.rpc.BadRequest",
      "fieldViolations": [
        {
          "field": "filter",
          "description": "Unsupported operator."
        }
      ]
    }
  ]
}
```

## Practical Rule of Thumb

Ask these questions in order:

1. What canonical code best describes the failure?
2. What stable `ErrorInfo.reason` identifies it?
3. What dynamic values belong in `metadata`?
4. Is there a standard detail payload that structures this better?

If the answer to the last question is yes, use both `ErrorInfo` and that detail payload.

## References

- [../reference/error-model.md](../reference/error-model.md)
- [../reference/error-reasons.md](../reference/error-reasons.md)
- [adr-error-model-and-google-rpc-compatibility.md](./adr-error-model-and-google-rpc-compatibility.md)
- [../reference/error-sources.md](../reference/error-sources.md)
- https://google.aip.dev/193
- https://cloud.google.com/storage/docs/reference/rpc/google.rpc
- https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.BadRequest
- https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.PreconditionFailure
- https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.ResourceInfo
- https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.RetryInfo
- https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Help
