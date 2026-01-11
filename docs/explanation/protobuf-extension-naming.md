# Protobuf Extension Naming

Naming conventions for protobuf extensions in trogon-proto.

## Rules

1. **Use wrapper messages** — extensible, avoids symbol conflicts
2. **Name extensions after target type** — `file`, `enum`, `enum_value`
3. **Use field numbers 100,000+** — private/unregistered range

## Wrapper Messages

A wrapper message contains the actual options. The extension just references the wrapper:

```protobuf
// ❌ Without wrapper — symbol conflicts if both use "namespace"
extend FileOptions { optional Namespace namespace = 1; }
extend EnumOptions { optional Namespace namespace = 2; }  // CONFLICT!

// ✅ With wrapper — no conflicts, extensible
message FileOptions { optional Namespace namespace = 1; }
message EnumOptions { optional Namespace namespace = 1; }

extend google.protobuf.FileOptions { optional FileOptions file = 870000; }
extend google.protobuf.EnumOptions { optional EnumOptions enum = 870001; }
```

**Benefits:**
- Internal fields (`namespace`) are scoped to wrapper — no symbol conflicts
- Add new fields to wrapper without consuming new extension numbers
- Follows buf.validate pattern (`MessageRules`, `FieldRules`, etc.)

## Nesting Inside Options

Nest type-specific messages inside their Options wrapper:

```protobuf
message EnumValueOptions {
  // Format is ONLY used by EnumValueOptions → nest it
  message Format {
    optional Namespace namespace = 1;
    string template = 2;
  }
  optional Format format = 1;
}
```

- `Format` becomes `trogon.uuid.v1.EnumValueOptions.Format`
- No conflict with a potential top-level `trogon.uuid.v1.Format`

`Namespace` stays in `namespace.proto` because it's shared by `FileOptions`, `EnumOptions`, and `EnumValueOptions.Format`.

## Current Structure

```
options.proto
├─ FileOptions           { namespace }           → file (870000)
├─ EnumOptions           { namespace }           → enum (870001)
└─ EnumValueOptions      { format }              → enum_value (870002)
   └─ Format (nested)    { namespace, template }

namespace.proto
└─ Namespace             { uuid | dns | url }    — shared
```

## Field Number Ranges

| Range | Usage |
|-------|-------|
| 1–999 | Reserved by Google |
| 1,000–99,999 | Public (register first) |
| **100,000+** | **Private (use this)** |

## References

- [buf.validate](https://github.com/bufbuild/protovalidate)
- [googleapis](https://github.com/googleapis/googleapis)
