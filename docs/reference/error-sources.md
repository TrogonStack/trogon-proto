# Error Sources

This page preserves the external references behind the protobuf error option and naming decisions in this repo.

Use this when you need the upstream proto definitions or public examples that informed the current option shape.

## Core Sources

### Protobuf Descriptor Sources

- `google.protobuf.descriptor.proto`:
  https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/descriptor.proto
- Protobuf custom options guide:
  https://protobuf.dev/programming-guides/proto3/#custom_options

### Google RPC Proto Sources

- `google.rpc.status.proto`:
  https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto
- `google.rpc.code.proto`:
  https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
- `google.rpc.error_details.proto`:
  https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto

### Google AIP

- AIP-192 Documentation:
  https://google.aip.dev/192
- AIP-193 Errors:
  https://google.aip.dev/193

## Public Documentation Examples

### Reason and ErrorInfo Examples

- VMware Engine `ErrorReason`:
  https://docs.cloud.google.com/vmware-engine/docs/reference/rest/v1/ErrorReason
- Google common protos `ErrorReason`:
  https://docs.cloud.google.com/dotnet/docs/reference/Google.Api.CommonProtos/latest/Google.Api.ErrorReason
- `ErrorInfo` field reference:
  https://cloud.google.com/network-connectivity/docs/reference/networkconnectivity/rest/v1/ErrorInfo

### Generated API Documentation Example

- Street View Publish generated docs:
  https://cloud.google.com/go/docs/reference/cloud.google.com/go/streetview/latest/publish/apiv1/publishpb

## Related Repo Docs

- [Document Errors in Proto](../how-to/document-errors-in-proto.md)
- [Protobuf Extension Naming](../explanation/protobuf-extension-naming.md)
