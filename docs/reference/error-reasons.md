# Error Reasons

This document defines how trogon-proto should document service-specific `google.rpc.ErrorInfo.reason` values.

The goal is to give every service a stable, public reason catalog similar to Google's published `ErrorReason` references.

## Overview

Each service or package domain should maintain a reason catalog that answers:

- which `ErrorInfo.reason` values are valid
- which `domain` each reason belongs to
- which canonical `google.rpc.Code` values are normally paired with that reason
- which metadata keys may appear
- what the client is expected to do

## Reason Design Rules

Each reason should:

- be stable over time
- represent one logical error condition
- imply one broad client-handling path
- be unique within its `domain`
- use `UPPER_SNAKE_CASE`

Good reasons:

- `WIDGET_NOT_FOUND`
- `WIDGET_HAS_CHILDREN`
- `SNAPSHOT_EXPIRED`
- `PROJECTION_TIMEOUT`

Weak reasons:

- `DELETE_FAILED`
- `ERROR`
- `FAILED_PRECONDITION`

The canonical code is not a good replacement for a reason. The code is broad. The reason is specific.

## Domain Rules

Each reason belongs to exactly one `domain`.

The domain should:

- be globally unique
- usually identify the service that owns the error contract
- remain stable as long as the error contract remains stable

Recommended style:

- `widgets.trogonstack.com`
- `orders.trogonstack.com`
- `identity.trogonstack.com`

## Metadata Rules

Each `(reason, domain)` pair may define its own metadata keys.

Metadata keys should:

- use lowerCamelCase
- be at most 64 characters
- remain available once introduced
- distinguish required keys from optional keys
- include units in the key name when units matter

Good examples:

- `resourceName`
- `resourceType`
- `requestedVersion`
- `currentVersion`
- `zonesWithCapacity`

Avoid:

- opaque names
- multiple names for the same concept
- embedding units in values when the unit is part of the contract

## Catalog Format

Each service reason catalog should include, at minimum:

| Field | Meaning |
|-------|---------|
| `reason` | Stable `ErrorInfo.reason` value |
| `domain` | Owning `ErrorInfo.domain` |
| `default code` | Typical canonical `google.rpc.Code` |
| `metadata` | Required and optional metadata keys |
| `meaning` | What the error condition means |
| `client action` | What the caller should do next |

## Suggested Markdown Template

~~~md
## WIDGET_NOT_FOUND

- Domain: `widgets.trogonstack.com`
- Default code: `NOT_FOUND`
- Required metadata:
  - `resourceName`: full resource name
- Optional metadata:
  - `resourceType`: resource kind, usually `Widget`
- Meaning:
  The requested widget does not exist.
- Client action:
  Refresh local state, verify the identifier, or stop retrying.

Example:
~~~json
{
  "reason": "WIDGET_NOT_FOUND",
  "domain": "widgets.trogonstack.com",
  "metadata": {
    "resourceName": "widgets/123",
    "resourceType": "Widget"
  }
}
~~~
~~~

## Suggested Proto Enum Pattern

When the repo later introduces an actual reason enum, document each enum value directly in proto comments.

```proto
// Supported values for `google.rpc.ErrorInfo.reason` in the
// `widgets.trogonstack.com` error domain.
enum ErrorReason {
  ERROR_REASON_UNSPECIFIED = 0;

  // The requested widget does not exist.
  //
  // ErrorInfo:
  // - domain: `widgets.trogonstack.com`
  // - default code: `NOT_FOUND`
  // - metadata keys:
  //   - `resourceName` (required): full resource name
  //   - `resourceType` (optional): resource kind
  WIDGET_NOT_FOUND = 1;
}
```

That enum is documentation and stability guidance. The wire payload still carries a string `ErrorInfo.reason`.

## Versioning Rules

Once published:

- a reason should not silently change meaning
- a reason should not move to a different domain
- required metadata keys should not disappear
- optional metadata keys may be added

If the client-handling meaning changes, define a new reason instead of mutating the old one.

## Example Catalog Entries

### WIDGET_NOT_FOUND

- Domain: `widgets.trogonstack.com`
- Default code: `NOT_FOUND`
- Required metadata:
  - `resourceName`
- Optional metadata:
  - `resourceType`
- Meaning:
  The requested widget does not exist.
- Client action:
  Stop retrying and verify the resource identifier.

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

### WIDGET_HAS_CHILDREN

- Domain: `widgets.trogonstack.com`
- Default code: `FAILED_PRECONDITION`
- Required metadata:
  - `resourceName`
  - `childType`
- Meaning:
  The resource cannot be deleted because dependent resources still exist.
- Client action:
  Remove or reassign the child resources before retrying.

Example:

```json
{
  "reason": "WIDGET_HAS_CHILDREN",
  "domain": "widgets.trogonstack.com",
  "metadata": {
    "resourceName": "widgets/123",
    "childType": "Part"
  }
}
```

### PROJECTION_TIMEOUT

- Domain: `widgets.trogonstack.com`
- Default code: `UNAVAILABLE`
- Required metadata:
  - `requestedVersion`
  - `currentVersion`
- Optional metadata:
  - `attempts`
  - `elapsedMs`
- Meaning:
  The read model did not catch up in time to satisfy the requested consistency level.
- Client action:
  Retry with backoff or relax the consistency requirement.

## References

- [error-model.md](./error-model.md)
- [error-sources.md](./error-sources.md)
- https://google.aip.dev/193
- https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo
- https://docs.cloud.google.com/vmware-engine/docs/reference/rest/v1/ErrorReason
- https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason
