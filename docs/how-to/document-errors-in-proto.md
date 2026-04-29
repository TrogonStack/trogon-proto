# Document Errors in Proto

This guide explains how to document an RPC's error contract when using the Google gRPC error model.

Use this when adding a new RPC or tightening the contract of an existing one.

## Goal

Every RPC should communicate:

- which canonical status codes it may return
- which stable machine-readable reasons matter
- which dynamic values the client may rely on

## Step 1: Choose the Canonical Status Code

Pick the narrowest `google.rpc.Code` that describes the failure class.

Common choices:

- `INVALID_ARGUMENT`: malformed request or invalid field shape
- `NOT_FOUND`: missing resource
- `ALREADY_EXISTS`: creation conflict
- `PERMISSION_DENIED`: caller lacks authorization
- `FAILED_PRECONDITION`: valid request, invalid current system state
- `UNAVAILABLE`: transient backend or timing issue

If you are unsure between state and transience, re-check the gRPC status code guidance before inventing a custom rule.

## Step 2: Decide Whether a Stable Reason Is Needed

Define or reuse an `ErrorInfo.reason` when:

- clients need to distinguish multiple failures with the same code
- the error contract should survive message text changes
- the error will appear in multiple RPCs
- the error should be documented in a shared catalog

Reuse an existing reason when the client action is the same.

Create a new reason when the client action differs.

## Step 3: Choose the Domain

Set `ErrorInfo.domain` to the service that owns the contract.

Examples:

- `widgets.trogonstack.com`
- `orders.trogonstack.com`

Do not use a different domain unless a different subsystem truly owns the error semantics.

## Step 4: Define Metadata Keys

List the dynamic data that a client may need in order to handle the error programmatically.

Good metadata data points:

- resource identifiers
- expected versus actual version values
- capacity or limit context
- consistency or retry context

For each key, decide:

- required or optional
- exact name
- value format

## Step 5: Prefer Standard Detail Payloads Where They Fit

Before putting everything into `ErrorInfo.metadata`, decide whether a standard detail message is better.

Use:

- `BadRequest` for field violations
- `PreconditionFailure` for invariant failures
- `ResourceInfo` for resource context
- `RetryInfo` for retry timing
- `Help` for troubleshooting links

Keep `ErrorInfo` even when those are present.

## Step 6: Document the RPC Comment

Add a leading comment on the RPC that lists the important canonical codes and, when useful, the stable reasons.

Example:

```proto
// Deletes a widget.
//
// Returns the following error codes:
// - `NOT_FOUND` with `ErrorInfo.reason = WIDGET_NOT_FOUND` if the widget does not exist.
// - `FAILED_PRECONDITION` with `ErrorInfo.reason = WIDGET_HAS_CHILDREN` if the widget still has child resources.
rpc DeleteWidget(DeleteWidgetRequest) returns (google.protobuf.Empty);
```

## Step 7: Update the Shared Reason Catalog

If the RPC introduces a new reason:

- add it to the service reason catalog
- document the domain
- document required and optional metadata keys
- add an example payload

See [../reference/error-reasons.md](../reference/error-reasons.md).

## Step 8: Add a Concrete Example

When the contract is subtle, include a JSON example in the catalog or explanation docs.

Example:

```json
{
  "code": 9,
  "message": "Widget cannot be deleted while child resources still exist.",
  "details": [
    {
      "@type": "type.googleapis.com/google.rpc.ErrorInfo",
      "reason": "WIDGET_HAS_CHILDREN",
      "domain": "widgets.trogonstack.com",
      "metadata": {
        "resourceName": "widgets/123",
        "childType": "Part"
      }
    },
    {
      "@type": "type.googleapis.com/google.rpc.PreconditionFailure",
      "violations": [
        {
          "type": "DEPENDENCY",
          "subject": "widgets/123",
          "description": "Widget still has child resources."
        }
      ]
    }
  ]
}
```

## Checklist

Use this checklist before finalizing an RPC error contract:

- The canonical code is the narrowest correct code.
- `ErrorInfo.reason` is stable and specific.
- `ErrorInfo.domain` is globally unique and owned by the service.
- Metadata keys are documented and named consistently.
- Standard detail payloads are used where they add structure.
- The RPC comment lists the important error codes.
- The shared reason catalog is updated.

## Common Mistakes

- Using only message text and skipping `ErrorInfo`
- Reusing the same reason for different client actions
- Putting validation details into metadata instead of `BadRequest`
- Using `FAILED_PRECONDITION` where `INVALID_ARGUMENT` is more accurate
- Using `UNAVAILABLE` for permanent business-rule failures
- Omitting metadata that appears in the human-readable message

## Example End-to-End Pattern

### Create RPC

```proto
// Creates a widget.
//
// Returns the following error codes:
// - `INVALID_ARGUMENT` with `ErrorInfo.reason = INVALID_WIDGET_SPEC` if the request is malformed.
// - `ALREADY_EXISTS` with `ErrorInfo.reason = WIDGET_ALREADY_EXISTS` if the widget ID is already in use.
rpc CreateWidget(CreateWidgetRequest) returns (Widget);
```

### Delete RPC

```proto
// Deletes a widget.
//
// Returns the following error codes:
// - `NOT_FOUND` with `ErrorInfo.reason = WIDGET_NOT_FOUND` if the widget does not exist.
// - `FAILED_PRECONDITION` with `ErrorInfo.reason = WIDGET_HAS_CHILDREN` if the widget still has child resources.
rpc DeleteWidget(DeleteWidgetRequest) returns (google.protobuf.Empty);
```

## References

- [../reference/error-model.md](../reference/error-model.md)
- [../reference/error-reasons.md](../reference/error-reasons.md)
- [../reference/error-sources.md](../reference/error-sources.md)
- https://google.aip.dev/192
- https://google.aip.dev/193
- https://grpc.io/docs/guides/status-codes/
