# Protobuf Extension Naming

Guidelines for creating protobuf extensions in trogon-proto. Follow these conventions when adding new extensions.

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

## Structure Template

When creating a new extension at `trogon/<feature>/<version>/options.proto`:

```text
trogon/<feature>/<version>/options.proto
├─ FileOptions           { ... }  → file (8700XX)
├─ MessageOptions        { ... }  → message (8700XX)
├─ EnumOptions           { ... }  → enum (8700XX)
├─ EnumValueOptions      { ... }  → enum_value (8700XX)
├─ FieldOptions          { ... }  → field (8700XX)
└─ <Nested Types>                 — type-specific, nested inside parent Options
```

**Pick the descriptor types you need:**
- `FileOptions` - file-level options (e.g., module namespace)
- `MessageOptions` - message-level options (e.g., validation rules)
- `EnumOptions` - enum-level options (e.g., allowed values)
- `EnumValueOptions` - enum value-level options (e.g., code generation hints)
- `FieldOptions` - field-level options (e.g., format constraints)

**Field number selection:**
- Start at next available number in 870000+ range
- Each wrapper gets its own extension number
- Internal fields within wrappers can reuse numbers (they're scoped)

## Field Number Registry

Claim your extension numbers here to prevent conflicts:

| Number | Extension | Descriptor Type |
|--------|-----------|-----------------|
| 870000 | trogon.uuid.v1 | google.protobuf.FileOptions |
| 870001 | trogon.uuid.v1 | google.protobuf.EnumOptions |
| 870002 | trogon.uuid.v1 | google.protobuf.EnumValueOptions |
| 870010 | trogon.object_id.v1alpha1 | google.protobuf.EnumValueOptions |
| 870011–870999 | *Available* | — |

## Examples in This Repo

```text
trogon/uuid/v1/options.proto
├─ FileOptions           { namespace }           → file (870000)
├─ EnumOptions           { namespace }           → enum (870001)
└─ EnumValueOptions      { format }              → enum_value (870002)
   └─ Format (nested)    { namespace, template }

trogon/object_id/v1alpha1/options.proto
└─ EnumValueOptions      { object_type, separator } → enum_value (870010)
```

## Field Number Ranges

| Range | Usage |
|-------|-------|
| 1–999 | Reserved by Google |
| 1,000–99,999 | Public (register first) |
| **100,000+** | **Private (use this)** |

## Creating a New Extension

1. **Choose a feature name and version**: `trogon/<feature>/<version>/`
   - Use semantic versioning: `v1`, `v2`, or alpha/beta: `v1alpha1`, `v1beta1`
   - Example: `trogon/validation/v1/`, `trogon/codegen/v1alpha1/`

2. **Create `options.proto`**: `proto/trogon/<feature>/<version>/options.proto`
   ```protobuf
   syntax = "proto3";
   package trogon.<feature>.<version>;

   import "elixirpb.proto";
   import "google/protobuf/descriptor.proto";

   option (elixirpb.file).module_prefix = "TrogonProto.<Feature>.<Version>";
   ```

3. **Define wrapper messages** for each descriptor type you need:
   ```protobuf
   message EnumValueOptions {
     string my_field = 1;
     optional string my_optional_field = 2;
   }
   ```

4. **Extend the Google descriptor** with your wrapper:
   ```protobuf
   extend google.protobuf.EnumValueOptions {
     optional EnumValueOptions enum_value = 8700XX;  // pick next available
   }
   ```

5. **Document in this file** - add your extension to the "Examples" section

6. **Update field number registry** - claim your number(s) in this doc to prevent conflicts

## References

- [buf.validate](https://github.com/bufbuild/protovalidate)
- [googleapis](https://github.com/googleapis/googleapis)
