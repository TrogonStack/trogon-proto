# Document NATS Micro Services

Use `trogon.nats.micro.v1alpha1` options when a protobuf service should also describe its NATS Services contract.

These annotations describe the proto contract. They do not create a runtime service by themselves, and they do not require a Go generator. A generator can map the same descriptor metadata onto `github.com/nats-io/nats.go/micro`, another NATS client, or a non-Go runtime.

## Annotate a Service

```protobuf
syntax = "proto3";
package acme.orders.v1;

import "trogon/nats/micro/v1alpha1/options.proto";

service OrderService {
  option (trogon.nats.micro.v1alpha1.service) = {
    version: "1.0.0"
    description: "Order command and query endpoints"
    metadata: { key: "owner" value: "orders" }
  };

  rpc GetOrder(GetOrderRequest) returns (Order) {
    option (trogon.nats.micro.v1alpha1.method) = {
      metadata: { key: "operation" value: "read" }
    };
  }
}
```

The service discovery name defaults to `OrderService`, and the endpoint name defaults to the protobuf RPC method name.

`version`, `description`, and `metadata` are service discovery metadata. They are not extra subject segments.

## Configure Discovery Metadata

Use service options to describe NATS service discovery metadata.

- `metadata` is copied into service discovery metadata.

Use method options only for endpoint-specific metadata.

- `metadata` is copied into endpoint discovery metadata.

## Signal Payload Format

Use `content_type` to restrict the NATS message payload format. This is the per-message NATS `Content-Type` header, not NATS service or endpoint metadata. Generated clients send one concrete `Content-Type` per message, and generated handlers use the incoming header to decode request and response payloads. If unset, generators support both protobuf and protobuf JSON.

`CONTENT_TYPE_PROTOBUF` maps to `application/protobuf`. `CONTENT_TYPE_JSON` maps to `application/json`.

`CONTENT_TYPE_UNSPECIFIED` is the default and means both `CONTENT_TYPE_PROTOBUF` and `CONTENT_TYPE_JSON` are supported. It does not put both content types on one message.

When the incoming message has no `Content-Type` header, generated handlers treat it as `CONTENT_TYPE_PROTOBUF`.

Set `content_type` only to restrict the service to one format.

## Error Contract

Use the existing `trogon.error.v1alpha1` annotations for structured error details.

Do not encode structured error details in NATS metadata. Typed details belong in `google.rpc.Status.details` or Trogon-owned error payload messages.

## Runtime Boundaries

Keep runtime and deployment behavior out of the proto contract unless it is stable service semantics.

Do not model these in the first NATS micro proto contract:

- queue group selection
- credentials or authorization
- retry policy
- pending limits
- JetStream KV or object-store side effects
- process topology

Those settings belong to runtime configuration until their semantics are stable enough to become part of the protobuf contract.
