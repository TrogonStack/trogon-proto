# Document Errors in Proto

This guide shows the intended protobuf shape for typed error payload messages that use `trogon.error.v1alpha1`.

## Example

```proto
import "trogon/error/v1alpha1/code.proto";
import "trogon/error/v1alpha1/options.proto";

message ResourceAvailabilityError {
  option (trogon.error.v1alpha1.message).template = {
    domain: "compute.googleapis.com"
    reason: "RESOURCE_AVAILABILITY"
    message: "Requested resources are unavailable."
    code: RESOURCE_EXHAUSTED
  };

  string zone = 1;
  string vm_type = 2;
  string attachment = 3;
  string zones_with_capacity = 4;
}
```

## Notes

- `domain`, `reason`, `message`, and `code` use explicit presence in proto3.
- `template` also has presence because it is a message field.
- Dynamic payload fields are currently scalar `string` fields.

## References

- [../../proto/trogon/error/v1alpha1/code.proto](../../proto/trogon/error/v1alpha1/code.proto)
- [../../proto/trogon/error/v1alpha1/options.proto](../../proto/trogon/error/v1alpha1/options.proto)
