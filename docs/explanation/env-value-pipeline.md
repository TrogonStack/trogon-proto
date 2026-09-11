# Environment Value Pipeline

## Overview

The `trogon.env.v1alpha1` package annotates config fields with the metadata a
code generator needs to load them from the environment. `EnvVarOption.steps` is
that metadata's core: an ordered list of operations that derives the field value
from the environment variable's text.

## Problem Statement

Environment variables are text. A `bytes` field therefore has no single obvious
reading of its value, and operators frequently supply the same key in more than
one form. A 32 byte HMAC-SHA256 signing key shows up either base64 encoded or as
32 raw bytes, so loaders end up hand written per field:

```go
var candidates [][]byte
if decoded, err := base64.StdEncoding.DecodeString(raw); err == nil {
    candidates = append(candidates, decoded)
}
candidates = append(candidates, []byte(raw))

for _, candidate := range candidates {
    if len(candidate) == 32 {
        return candidate, nil
    }
}
return nil, fmt.Errorf("TOKEN_SIGNING_KEY must decode to a 32 byte key")
```

Three separate concerns are collapsed into that loader: how to decode, which
decoding to trust, and what shape is acceptable. Written by hand, every field
restates all three, and each restatement is a chance to leak the value into a
crash message.

## Order Is the Contract

The annotation is a list, not a set of keys, because the operations are a
procedure and a reader should not have to consult prose to learn what runs
first:

```protobuf
bytes token_signing_key = 1 [(trogon.env.v1alpha1.field).env_var = {
  visibility: VISIBILITY_SECRET
  steps: [
    {trim: {unicode_whitespace: {}}},
    {decode: {
      any_of: [
        {base64: {alphabet: ALPHABET_STANDARD, padding: PADDING_REQUIRED}},
        {utf8: {}}
      ]
      accept: {byte_size: {exact: 32}}
    }}
  ]
}];
```

Read top to bottom: trim the surrounding whitespace, then read the value as
base64 or as raw bytes, requiring 32 bytes either way.

## The Steps

| step | does | valid on |
|---|---|---|
| `split` | turns the whole value into elements | repeated fields only |
| `trim` | removes leading and trailing characters | text |
| `decode` | turns text into bytes | `bytes` and `string` fields |
| `require` | validates, produces no new value | `bytes` and `string` fields |

`require` is limited to `bytes` and `string` fields because `byte_size` is the
only constraint `Constraints` defines today. The step generalizes when
`Constraints` grows.

`split` is the only step that changes how many values are in play. Steps before
it see the whole environment value; steps after it run once per element. A
non-repeated field has no `split`, so every step sees the single value. That is
the whole of the scope rule, and it is why `byte_size` on a repeated field always
means "each element", never "the whole variable".

```protobuf
repeated bytes rotation_keys = 2 [(trogon.env.v1alpha1.field).env_var = {
  steps: [
    {split: {delimiter: ","}},
    {trim: {unicode_whitespace: {}}},
    {decode: {any_of: [{hex: {}}]}},
    {require: {byte_size: {exact: 32}}}
  ]
}];
```

## `steps` Is a Sequence, `any_of` Is a Choice

Both are `repeated`, and they mean opposite things:

- **`steps`** runs every entry, in order. Each step's output is the next step's
  input.
- **`decode.any_of`** runs at most one entry. Every candidate receives the same
  input, and the first whose output satisfies `accept` is the value.

The quantifier is in the field name for exactly this reason. A bare list is a
sequence; alternatives always sit under `any_of`.

`accept` is required whenever `any_of` holds more than one candidate, because a
choice needs something to discriminate on. It also explains why the expected
size is written once rather than twice: the rule that picks the reading is the
same rule that validates it. Use a standalone `require` step when there is
nothing to choose between.

## Why a Choice Needs a Rule at All

"Try base64, otherwise take the raw value" cannot be expressed as "first
candidate that decodes without error". A 32 character ASCII key is very often
valid base64 as well, and decodes cleanly to 24 bytes. A first-success rule
would silently accept those 24 bytes and the service would boot with the wrong
key. The expected size is what disambiguates, so it belongs to the choice.

## Defaults Follow the Same Pipeline

`default_value` substitutes for the environment value at the start of the
pipeline and then runs through every step. It is written exactly as an operator
would write the variable, and no step treats it specially:

```protobuf
repeated int32 allowed_ports = 3 [(trogon.env.v1alpha1.field).env_var = {
  default_value: "8080,8081"
  steps: [
    {split: {delimiter: ","}},
    {trim: {unicode_whitespace: {}}}
  ]
}];
```

Absent `ALLOWED_PORTS`, that yields `[8080, 8081]`.

## Failure Messages Never Carry the Value

A generator may report the field name, the environment variable name, the
candidates it attempted, and the observed byte size. It must not include the
value or any part of it unless `visibility` is `VISIBILITY_PLAINTEXT`. The
observed size is enough to diagnose a wrong key, and `visibility` already
declares whether the value is safe to print.

## Migrating From `split_delimiter` and `trim`

Those two fields are deprecated and kept for one release. Setting them together
with `steps` is a schema error, so migrate a field in one move:

```protobuf
// before
split_delimiter: ","
trim: {unicode_whitespace: {}}

// after
steps: [
  {split: {delimiter: ","}},
  {trim: {unicode_whitespace: {}}}
]
```

The step form also fixes a gap in the old shape: `trim` used to require
`split_delimiter`, so a singular field could not be trimmed at all. That made
`TOKEN_SIGNING_KEY=$(cat key.b64)` unloadable, because the trailing newline
broke base64 decoding and nothing could strip it. A `trim` step has no
dependency on splitting.

## Scope

`Constraints` covers what choice resolution and load-time validation need, and
nothing more. It is deliberately not a general validation vocabulary: no
patterns, no numeric ranges, no cross-field rules. Those belong to whatever
validates the message as a whole, and adding them here would create a second
dialect competing with it.

`map` fields are undefined. Loading `FOO="a:1,b:2"` into a `map<string, string>`
needs two splits at different types, entries out of the whole value and then a
key/value pair out of each entry, which is a step type that does not exist yet.
Generators reject `steps` on map fields rather than improvise.
