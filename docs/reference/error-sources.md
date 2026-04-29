# Error Sources

This page preserves the primary sources and example references behind the repo's error-model guidance.

Use this when you need to revisit the external evidence for:

- the canonical ADR-shaped error model
- Google RPC compatibility behavior
- proto documentation guidance
- public error catalog patterns
- OpenAPI generation behavior
- runtime adapter design

## Core Sources

### Straw Hat ADR

- Straw Hat ADR error specification:
  https://straw-hat-team.github.io/adr/adrs/0129349218/README.html

### Google AIP

- AIP-192 Documentation:
  https://google.aip.dev/192
- AIP-193 Errors:
  https://google.aip.dev/193
- AIP-194 Automatic retry configuration:
  https://google.aip.dev/194
- AIP-4221 Client library retry configuration:
  https://google.aip.dev/client-libraries/4221

### Google RPC Proto Sources

- `google.rpc.Status`:
  https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- `google.rpc.Code`:
  https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
- `google.rpc.ErrorInfo` and standard detail payloads:
  https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto

## Public Documentation Examples

### Per-RPC Error Code Documentation

- Street View Publish generated docs:
  https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/0.2.1/publish/apiv1/publishpb
- Street View Publish generated docs, latest:
  https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/latest/publish/apiv1/publishpb

These are useful examples of the Google pattern where method docs explicitly list the canonical error codes a method may return.

### Reason Catalogs

- VMware Engine `ErrorReason`:
  https://docs.cloud.google.com/vmware-engine/docs/reference/rest/v1/ErrorReason
- Google common protos `ErrorReason`:
  https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason

These are the main examples behind the repo's reason-catalog guidance.

### ErrorInfo Reference

- `ErrorInfo` field reference:
  https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo
- `google.rpc` package reference:
  https://cloud.google.com/storage/docs/reference/rpc/google.rpc
- `google.rpc` package reference, alternate:
  https://cloud.google.com/bigquery/docs/reference/migration/rpc/google.rpc

## OpenAPI Generation References

- gRPC-Gateway OpenAPI customization:
  https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/customizing_openapi_output/
- gRPC-Gateway response schema references:
  https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/using_ref_with_responses/

These links matter for the repo's conclusion that OpenAPI should come from the canonical proto model rather than from raw `google.rpc.Status` alone.

## Runtime Adapter References

### Elixir

- `Trogon.Error`:
  https://hexdocs.pm/trogon_error/Trogon.Error.html
- `Trogon.Error.Metadata`:
  https://hexdocs.pm/trogon_error/Trogon.Error.Metadata.html
- `Trogon.Error.MetadataValue`:
  https://hexdocs.pm/trogon_error/Trogon.Error.MetadataValue.html

### Go

- `trogonerror` package docs:
  https://pkg.go.dev/github.com/TrogonStack/trogonerror

## Related Repo Docs

- [Error Model](./error-model.md)
- [Error Reasons](./error-reasons.md)
- [Document Errors in Proto](../how-to/document-errors-in-proto.md)
- [Generate OpenAPI and trogon_error from Proto](../how-to/generate-openapi-and-trogon-error-from-proto.md)
- [ADR Error Model and Google RPC Compatibility](../explanation/adr-error-model-and-google-rpc-compatibility.md)
- [Error Detail Selection](../explanation/error-detail-selection.md)
