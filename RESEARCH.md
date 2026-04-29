# Research: Documenting Google-Style gRPC Errors in Proto

Verified on 2026-04-28.

This document records the research around one specific question:

> If a service uses `google.rpc.Status` and `google.rpc.ErrorInfo`, how should it document which errors an RPC may return, and how can `reason`, `domain`, and `metadata` be documented in the proto itself?

## Short Answer

The Google/AIP model separates the problem into two layers:

1. Runtime errors are standardized with `google.rpc.Status`, canonical `google.rpc.Code`, and `google.rpc.ErrorInfo` in `Status.details`.
2. Error contracts are documented with:
   - RPC comments for method-specific canonical error codes.
   - A shared reason catalog for stable `ErrorInfo.reason` values, their `domain`, and their `metadata` keys.

What the research did **not** find is an AIP-standard, machine-readable proto annotation that declares:

- which errors each RPC may return
- which `ErrorInfo.reason` values are allowed for that RPC
- which metadata keys are guaranteed for each reason

That means the Google pattern is mostly:

- comments for per-RPC documentation
- enum-like catalogs for stable reasons
- optional project-specific custom options if machine-readable contracts are needed

## Research Questions

The research focused on five questions:

1. What is the standard Google/AIP runtime error model?
2. Does AIP require documenting common errors in proto comments?
3. How do Google APIs document method-specific errors in practice?
4. How do Google APIs document stable `ErrorInfo.reason` values, `domain`, and `metadata` keys?
5. Is there a standard proto-native way to make those contracts machine-readable?

## Finding 1: The Standard Runtime Error Model Is `Status` + `Code` + `ErrorInfo`

The core runtime model is defined by `google.rpc.Status`, `google.rpc.Code`, and `google.rpc.ErrorInfo`.

### What AIP-193 says

AIP-193 requires services to:

- return `google.rpc.Status` for API errors
- use canonical `google.rpc.Code` values
- include `google.rpc.ErrorInfo` in `Status.details`

It also defines key constraints for `ErrorInfo`:

- `reason` is stable within a `domain`
- `domain` is the logical grouping for the reason, typically the service name
- `metadata` contains machine-readable dynamic context
- metadata keys must remain stable over time once introduced for a `(reason, domain)` pair

### Why this matters

This means the machine-readable identity of an error is not only the status code. It is the combination of:

- `google.rpc.Code`
- `google.rpc.ErrorInfo.reason`
- `google.rpc.ErrorInfo.domain`
- stable expectations around `metadata`

### Sources

- AIP-193: https://google.aip.dev/193
- `google.rpc.Status` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- `google.rpc.Code` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
- `google.rpc.ErrorInfo` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto
- `ErrorInfo` field reference: https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo

## Finding 2: AIP-192 Expects Common Errors to Be Documented in Proto Comments

AIP-192 is the Google API documentation guidance for proto comments.

Its documentation checklist explicitly asks authors to document:

- what happens if something fails
- common errors that may break the operation

This is the strongest official guidance for documenting per-RPC errors in proto itself.

### Implication

If an RPC commonly returns:

- `NOT_FOUND`
- `FAILED_PRECONDITION`
- `PERMISSION_DENIED`
- `UNAVAILABLE`

those should be called out in the RPC's leading comment, along with the condition that triggers them.

### Source

- AIP-192: https://google.aip.dev/192

## Finding 3: Google API Docs Commonly Enumerate Canonical Error Codes Per RPC

The most direct example found in Google-generated docs is the Street View Publish API reference. Its method docs include text like:

- "This method returns the following error codes:"
- followed by a list of canonical `google.rpc.Code` values and the condition for each one

Examples visible in the generated reference include methods such as:

- `UpdatePhoto`
- `CreatePhoto`
- `GetPhoto`
- `DeletePhoto`
- `DeletePhotoSequence`

### Implication

This is the clearest evidence that Google documents method-specific errors primarily as:

- RPC comments in the proto
- surfaced into generated API docs

### Source

- Street View Publish generated docs: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/0.2.1/publish/apiv1/publishpb

## Finding 4: Google Also Publishes Reason Catalogs for `ErrorInfo.reason`

Per-RPC error code lists are only half of the story. Google also publishes stable reason catalogs for the machine-readable `ErrorInfo.reason` field.

Two useful examples were found.

### Example A: VMware Engine `ErrorReason`

The VMware Engine docs publish a dedicated `ErrorReason` reference page.

That page:

- defines supported values for `google.rpc.ErrorInfo.reason`
- scopes them to a single error domain: `vmwareengine.googleapis.com`
- gives a prose explanation for each reason
- shows a sample `ErrorInfo` object for each reason
- shows the `domain`
- shows example `metadata` payloads, including which keys may appear

This is the closest Google example to a public "error contract catalog".

### Source

- VMware Engine `ErrorReason`: https://docs.cloud.google.com/vmware-engine/docs/reference/rest/v1/ErrorReason

### Example B: Google API Common Protos `ErrorReason`

Google also publishes a public `ErrorReason` enum for the `googleapis.com` domain in the common protos client docs.

That documentation states:

- the enum defines supported `google.rpc.ErrorInfo.reason` values for the `googleapis.com` domain
- some metadata keys are shared across the domain, such as `service`
- other metadata keys vary by specific reason
- each reason includes an example `ErrorInfo` payload

This is especially important because it shows a documented pattern where:

- the reason catalog is shared at the domain level
- metadata conventions can be partially domain-wide and partially reason-specific

### Source

- `Google.Api.ErrorReason` docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason

## Finding 5: `ErrorInfo.metadata` Is Deliberately Flexible

The hardest part to document in proto is `metadata`.

The reason is structural:

- `google.rpc.ErrorInfo.metadata` is `map<string, string>`
- it is not a typed message with fixed fields

That means protobuf itself does not give a built-in, strongly typed schema for:

- which metadata keys exist for a given reason
- which keys are required
- which RPCs may emit which metadata shape

### What AIP-193 requires

AIP-193 does still impose real contract expectations:

- metadata keys should be stable once introduced for a `(reason, domain)` pair
- request-specific information used in human-readable messages must also appear in metadata
- keys should be consistently named across reasons within a domain

### Implication

The contract is real, but it is documented behavior, not wire-level protobuf typing.

### Sources

- AIP-193: https://google.aip.dev/193
- `ErrorInfo` field reference: https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo

## Finding 6: No Standard AIP/Google Method Annotation Was Found for Per-RPC Error Contracts

The research did not find a standard Google/AIP proto option that lets authors declare, in a machine-readable way:

- the canonical error codes an RPC may return
- the allowed `ErrorInfo.reason` values for that RPC
- the allowed metadata keys for each reason

This does **not** mean a project cannot do it. Protobuf supports custom options, and projects can define their own method annotations. It does mean that this is not the default Google AIP pattern found in the research.

### Practical conclusion

If this repo wants machine-readable error contracts inside the proto, it will need a project-owned custom option.

## What Can Be Documented in Proto Itself

Based on the research, the closest proto-native shape to Google's public documentation style is:

1. A shared `ErrorReason` enum for the service or package.
2. Comments on each enum value documenting:
   - what the reason means
   - the fixed `domain`
   - the metadata keys that may appear
   - which keys are required or optional
   - a sample `ErrorInfo` payload
3. RPC comments listing:
   - canonical `google.rpc.Code` values
   - the applicable `ErrorReason` values
4. Optional documentation-only companion metadata messages.

### Example

```proto
syntax = "proto3";

package trogon.widgets.v1;

// Supported values for `google.rpc.ErrorInfo.reason` in the
// `widgets.trogonstack.com` error domain.
enum ErrorReason {
  // Do not use this default value.
  ERROR_REASON_UNSPECIFIED = 0;

  // The requested widget does not exist.
  //
  // ErrorInfo:
  // - domain: `widgets.trogonstack.com`
  // - metadata keys:
  //   - `resourceName` (required): full resource name, e.g. `widgets/123`
  //   - `resourceType` (optional): resource kind, e.g. `Widget`
  //
  // Example:
  // {
  //   "reason": "WIDGET_NOT_FOUND",
  //   "domain": "widgets.trogonstack.com",
  //   "metadata": {
  //     "resourceName": "widgets/123",
  //     "resourceType": "Widget"
  //   }
  // }
  WIDGET_NOT_FOUND = 1;

  // The widget cannot be deleted because it still has child resources.
  //
  // ErrorInfo:
  // - domain: `widgets.trogonstack.com`
  // - metadata keys:
  //   - `resourceName` (required): full resource name
  //   - `childType` (required): child resource kind
  WIDGET_HAS_CHILDREN = 2;
}

// Documentation-only schema for the metadata used by WIDGET_NOT_FOUND.
// The wire format still uses google.rpc.ErrorInfo.metadata.
message WidgetNotFoundErrorMetadata {
  string resource_name = 1;
  string resource_type = 2;
}

// Deletes a widget.
//
// Returns the following error codes:
// - `NOT_FOUND` with `ErrorInfo.reason = WIDGET_NOT_FOUND`
// - `FAILED_PRECONDITION` with `ErrorInfo.reason = WIDGET_HAS_CHILDREN`
rpc DeleteWidget(DeleteWidgetRequest) returns (google.protobuf.Empty);
```

## What Proto Cannot Express Cleanly with Standard Google Types

Using only the standard Google error model, proto does **not** strongly encode:

- that `DeleteWidget` may only return `WIDGET_NOT_FOUND` and `WIDGET_HAS_CHILDREN`
- that `WIDGET_NOT_FOUND` always carries `resourceName`
- that `WIDGET_HAS_CHILDREN` always carries `childType`
- that the `domain` for all widget reasons is `widgets.trogonstack.com`

Those contracts can be documented in proto comments, but they are not enforced by the standard `google.rpc.ErrorInfo` message.

## If Machine-Readable Contracts Are Required

If the repo wants documentation and tooling to consume the error contract directly from proto, the project can define custom method options.

### Example direction

```proto
syntax = "proto3";

import "google/protobuf/descriptor.proto";

message ErrorMetadataField {
  string key = 1;
  bool required = 2;
  string description = 3;
}

message ErrorContract {
  string code = 1;
  string reason = 2;
  string domain = 3;
  repeated ErrorMetadataField metadata_fields = 4;
}

extend google.protobuf.MethodOptions {
  repeated ErrorContract errors = 870100;
}
```

Then each RPC could declare:

```proto
rpc DeleteWidget(DeleteWidgetRequest) returns (google.protobuf.Empty) {
  option (errors) = {
    code: "NOT_FOUND"
    reason: "WIDGET_NOT_FOUND"
    domain: "widgets.trogonstack.com"
    metadata_fields: {
      key: "resourceName"
      required: true
      description: "Full resource name of the missing widget."
    }
  };
  option (errors) = {
    code: "FAILED_PRECONDITION"
    reason: "WIDGET_HAS_CHILDREN"
    domain: "widgets.trogonstack.com"
    metadata_fields: {
      key: "resourceName"
      required: true
      description: "Full resource name of the widget."
    }
    metadata_fields: {
      key: "childType"
      required: true
      description: "Nested resource type blocking deletion."
    }
  };
}
```

### Trade-offs

Pros:

- machine-readable
- can power documentation generation
- can power linting or contract tests

Cons:

- not part of Google AIP
- requires custom tooling
- creates another schema to maintain
- still does not change the wire shape of `google.rpc.ErrorInfo`

## Recommended Documentation Pattern for This Repo

Based on the research, the most Google-aligned approach is:

1. Use `google.rpc.Status` and canonical `google.rpc.Code`.
2. Always include `google.rpc.ErrorInfo` in `Status.details`.
3. Define a shared `ErrorReason` catalog per service domain.
4. Document `domain` and `metadata` keys on each reason.
5. Document per-RPC canonical codes in RPC comments.
6. Use standard detail payloads when richer structure exists.

That last point matters:

- use `BadRequest` for field validation failures
- use `PreconditionFailure` for unmet invariants
- use `ResourceInfo` for resource identity and ownership context
- use `Help` when troubleshooting docs should be linked
- use `LocalizedMessage` when user-facing text matters

This keeps `ErrorInfo` focused on stable identity and lightweight machine-readable context, which is how AIP-193 positions it.

## Why the VMware Engine Pattern Is Useful

The VMware Engine `ErrorReason` page is a strong public model because it combines:

- stable reason identifiers
- domain scoping
- example payloads
- visible metadata expectations
- prose guidance for humans

In other words, it solves the discoverability problem even though `ErrorInfo.metadata` is only a string map on the wire.

For this repo, the nearest proto equivalent is:

- a shared `ErrorReason` enum
- very explicit enum value comments
- RPC comments that connect canonical codes to reasons

## Suggested Repo-Level Convention

If this repo adopts a convention, it could be:

1. One `errors.proto` per package or service domain.
2. One public `ErrorReason` enum per domain.
3. Every enum value comment must document:
   - `domain`
   - meaning
   - required metadata keys
   - optional metadata keys
   - sample `ErrorInfo`
4. Every RPC comment must document:
   - canonical status codes
   - applicable reasons
5. Rich structured details should prefer standard Google detail messages before adding more ad hoc metadata.

## Addendum: ADR Alignment, OpenAPI Generation, and trogon_error

The original research question focused on Google-style gRPC errors. A later review of Straw Hat ADR#0129349218 changes the preferred architecture for this repo.

### Finding 7: The Canonical Proto Model Should Follow the ADR

The ADR is explicit that the error specification is:

- data-oriented
- transport-independent
- not tied to a specific wire format

It also defines a broader error shape than Google RPC, including:

- `specversion`
- top-level `visibility`
- per-metadata-entry visibility
- recursive `causes`
- `subject`
- `id`
- `time`
- `source_id`

This means the strongest repo-level source of truth is not raw `google.rpc.Status`, but an ADR-shaped proto package that models the error specification directly.

### Sources

- ADR error specification: https://straw-hat-team.github.io/adr/adrs/0129349218/README.html

### Finding 8: Google RPC Should Be Treated as a Compatibility Transport

Google RPC maps well to several ADR concepts:

- `code`
- `domain`
- `reason`
- plain string metadata
- `Help`
- `DebugInfo`
- `LocalizedMessage`
- `RetryInfo`

However, Google RPC does not directly model:

- metadata entries as `{value, visibility}`
- top-level error visibility
- recursive `causes` as first-class domain errors
- the ADR filtering rules across trust boundaries

That makes Google RPC a good compatibility layer for gRPC, but not the best canonical error schema for the repo.

### Sources

- ADR error specification: https://straw-hat-team.github.io/adr/adrs/0129349218/README.html
- AIP-193: https://google.aip.dev/193
- `google.rpc.ErrorInfo` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto

### Finding 9: OpenAPI Should Be Generated from the Canonical ADR-Shaped Proto

If OpenAPI is generated from raw `google.rpc.Status`, the schema will usually be too weak because:

- `Status.details` uses `Any`
- `ErrorInfo.metadata` is a `map<string,string>`

OpenAPI generation becomes more precise if it is driven from a canonical ADR-shaped proto schema or from typed reason-specific error payload messages referenced by that canonical schema.

`grpc-gateway` can document explicit response schemas through OpenAPI proto options, which makes it a viable compatibility tool. It still benefits from having a stronger canonical schema to point to.

### Sources

- gRPC-Gateway OpenAPI customization: https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/customizing_openapi_output/
- gRPC-Gateway arbitrary response schema references: https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/using_ref_with_responses/

### Finding 10: trogon_error Generation Works Best from Typed Proto Error Payload Messages

The current `trogon_error` package already follows the ADR shape closely:

- template-level `domain`, `reason`, `code`, and `message`
- runtime metadata passed separately
- visibility-aware metadata entries

That means the best proto-to-Elixir generation pattern is:

1. Define one typed proto error payload message per stable reason.
2. Attach custom message options describing the static error template.
3. Generate a `Trogon.Error` integration that derives template information from proto.
4. Generate a `from_proto/1` constructor that flattens the typed error payload message into `Trogon.Error.Metadata`.

Important simplifications for now:

- no proto-side Elixir module naming hints
- only scalar string fields are supported for proto-to-`trogon_error` metadata
- list-like data should already be flattened into a string field in the proto contract

Naming guidance:

- if the same concept may also exist in non-error shapes, the public proto type should use an explicit `*Error` suffix
- avoid `*Metadata` in the public proto type name, because `metadata` is a transport/runtime concern rather than the domain concept

This is especially useful for examples such as:

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

### Sources

- `Trogon.Error`: https://hexdocs.pm/trogon_error/Trogon.Error.html
- `Trogon.Error.Metadata`: https://hexdocs.pm/trogon_error/Trogon.Error.Metadata.html
- `Trogon.Error.MetadataValue`: https://hexdocs.pm/trogon_error/Trogon.Error.MetadataValue.html

## Final Conclusion

The research points to a consistent answer:

- Google documents per-RPC errors with comments.
- Google documents stable machine-readable reasons with reason catalogs.
- Google does not appear to provide a standard proto annotation for full per-RPC error contracts.
- `ErrorInfo.metadata` is intentionally flexible, so its schema must be documented rather than inferred from the type.

If the goal is to stay close to Google/AIP style, the best fit is:

- RPC comments for error codes
- shared `ErrorReason` catalogs for `reason`, `domain`, and metadata keys

If the goal is stronger machine-readable contracts, that should be added as a project-specific custom option.

## Example Catalog

This section collects concrete reference pages that demonstrate the documentation patterns discussed above.

### A. Runtime Error Model Examples

These pages show the base error model and the standard error detail types.

- `google.rpc.Status` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- `google.rpc.Status` .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Status
- `google.rpc.Code` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
- `google.rpc.Code` .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Code
- `google.rpc.ErrorInfo` source: https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto
- `google.rpc.ErrorInfo` .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.ErrorInfo
- `google.rpc` package reference with `ErrorInfo`, `Status`, `BadRequest`, `Help`, `LocalizedMessage`, `PreconditionFailure`, `QuotaFailure`, `RequestInfo`, `ResourceInfo`, and `RetryInfo`: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
- `google.rpc` package reference with `ErrorInfo` examples: https://cloud.google.com/bigquery/docs/reference/migration/rpc/google.rpc

### B. `ErrorInfo` Payload Examples

These pages contain example `ErrorInfo` payloads with `reason`, `domain`, and `metadata`.

- `ErrorInfo` field reference with JSON examples and key rules: https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo
- `ErrorInfo` .NET docs with examples:
  - API disabled example: `reason = API_DISABLED`, `domain = googleapis.com`, metadata includes `resource` and `service`
  - stockout example: `reason = STOCKOUT`, `domain = spanner.googleapis.com`, metadata includes `availableRegions`
  - URL: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.ErrorInfo
- `google.rpc` package reference with `ErrorInfo` examples: https://cloud.google.com/bigquery/docs/reference/migration/rpc/google.rpc

### C. Per-RPC Error Code Documentation Examples

These pages show the Google pattern of listing canonical error codes directly on the method documentation.

- Street View Publish generated proto reference:
  - package-level generated docs: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/0.2.1/publish/apiv1/publishpb
  - latest generated proto docs: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/latest/publish/apiv1/publishpb
- Street View Publish client docs:
  - versioned client docs: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/0.2.1/publish/apiv1
  - latest client docs: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/latest/publish/apiv1

Useful concrete method examples visible in those docs:

- `CreatePhoto`: lists `INVALID_ARGUMENT`, `NOT_FOUND`, `RESOURCE_EXHAUSTED`
- `UpdatePhoto`: lists `PERMISSION_DENIED`, `INVALID_ARGUMENT`, `NOT_FOUND`, `UNAVAILABLE`
- `GetPhoto`: lists `PERMISSION_DENIED`, `NOT_FOUND`, `UNAVAILABLE`
- `DeletePhoto`: lists `PERMISSION_DENIED`, `NOT_FOUND`
- `DeletePhotoSequence`: lists `PERMISSION_DENIED`, `NOT_FOUND`, `FAILED_PRECONDITION`

### D. Reason Catalog Examples

These pages show the Google pattern of publishing a stable catalog of `ErrorInfo.reason` values.

- VMware Engine `ErrorReason`: https://docs.cloud.google.com/vmware-engine/docs/reference/rest/v1/ErrorReason
- Google common protos `ErrorReason`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason

What these examples demonstrate:

- a named reason catalog instead of free-form prose
- a fixed or well-known error domain
- reason-by-reason examples of `ErrorInfo`
- metadata conventions that are either shared by the domain or specific to each reason

### E. Standard Detail Payload Examples

These links are useful when deciding whether a piece of error data belongs in `ErrorInfo.metadata` or in a richer typed detail message.

- `BadRequest`:
  - package docs: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
  - .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.BadRequest
- `PreconditionFailure`:
  - package docs: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
  - .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.PreconditionFailure
- `ResourceInfo`:
  - package docs: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
  - .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.ResourceInfo
- `Help`:
  - package docs: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
  - .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Help
- `RetryInfo`:
  - package docs: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
  - .NET docs: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.RetryInfo
- `LocalizedMessage`:
  - package docs: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
  - PHP docs: https://docs.cloud.google.com/php/docs/reference/common-protos/latest/Rpc.LocalizedMessage

### F. gRPC Status Semantics Examples

These are not Google AIP docs, but they are still primary references for the underlying gRPC semantics.

- gRPC status codes guide: https://grpc.io/docs/guides/status-codes/
- gRPC error handling guide: https://grpc.io/docs/guides/error/

These are useful when deciding between codes such as:

- `FAILED_PRECONDITION`
- `ABORTED`
- `UNAVAILABLE`
- `INVALID_ARGUMENT`
- `OUT_OF_RANGE`

### G. Related AIP Reading

These are adjacent to the main question and help explain how clients should react to documented errors.

- AIP-194 Automatic retry configuration: https://google.aip.dev/194
- AIP-4221 Client library retry configuration: https://google.aip.dev/client-libraries/4221

## References

### Primary Google AIP Sources

- AIP-192 Documentation: https://google.aip.dev/192
- AIP-193 Errors: https://google.aip.dev/193
- AIP-194 Automatic retry configuration: https://google.aip.dev/194
- AIP-4221 Client library retry configuration: https://google.aip.dev/client-libraries/4221

### Primary Google Proto Sources

- `google.rpc.Status`: https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- `google.rpc.Status` raw source: https://raw.githubusercontent.com/googleapis/googleapis/master/google/rpc/status.proto
- `google.rpc.Code`: https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
- `google.rpc.Code` raw source: https://raw.githubusercontent.com/googleapis/googleapis/master/google/rpc/code.proto
- `google.rpc.ErrorInfo` and standard detail payloads: https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto
- `google.rpc.ErrorInfo` and standard detail payloads raw source: https://raw.githubusercontent.com/googleapis/googleapis/master/google/rpc/error_details.proto

### Google Cloud Reference Pages

- `ErrorInfo` field reference: https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo
- `google.rpc` package reference: https://cloud.google.com/bigquery/docs/reference/migration/rpc/google.rpc
- `google.rpc` package reference for common detail messages: https://cloud.google.com/storage/docs/reference/rpc/google.rpc
- `google.rpc` package reference with status and message details: https://docs.cloud.google.com/web-risk/docs/reference/rpc/google.rpc
- Street View Publish generated docs with per-method error code lists: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/0.2.1/publish/apiv1/publishpb
- Street View Publish generated docs with per-method error code lists, latest: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/latest/publish/apiv1/publishpb
- Street View Publish client docs, versioned: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/0.2.1/publish/apiv1
- Street View Publish client docs, latest: https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/latest/publish/apiv1
- VMware Engine `ErrorReason` catalog: https://docs.cloud.google.com/vmware-engine/docs/reference/rest/v1/ErrorReason
- Google common protos `ErrorReason` catalog: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason
- Straw Hat ADR error specification: https://straw-hat-team.github.io/adr/adrs/0129349218/README.html
- gRPC-Gateway OpenAPI customization: https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/customizing_openapi_output/
- gRPC-Gateway arbitrary response schema references: https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/using_ref_with_responses/
- `Trogon.Error`: https://hexdocs.pm/trogon_error/Trogon.Error.html
- `Trogon.Error.Metadata`: https://hexdocs.pm/trogon_error/Trogon.Error.Metadata.html
- `Trogon.Error.MetadataValue`: https://hexdocs.pm/trogon_error/Trogon.Error.MetadataValue.html

### Common Protos Client Docs

- `Google.Rpc` namespace overview: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc
- `Google.Rpc.Status`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Status
- `Google.Rpc.Code`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Code
- `Google.Rpc.ErrorInfo`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.ErrorInfo
- `Google.Rpc.BadRequest`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.BadRequest
- `Google.Rpc.PreconditionFailure`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.PreconditionFailure
- `Google.Rpc.ResourceInfo`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.ResourceInfo
- `Google.Rpc.Help`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.Help
- `Google.Rpc.RetryInfo`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Rpc.RetryInfo
- `Google.Api.ErrorReason`: https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason

### gRPC References

- gRPC status codes guide: https://grpc.io/docs/guides/status-codes/
- gRPC error handling guide: https://grpc.io/docs/guides/error/
