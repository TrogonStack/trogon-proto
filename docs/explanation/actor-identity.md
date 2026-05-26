# Actor Identity

The `trogon.actor.v1alpha1.Actor` message records who caused a domain event or action when consumers need both an actor type and an actor id.

At application boundaries, the caller may be represented as an authenticated principal, service account, workload identity, or system process. Domain events should not need to expose that IAM vocabulary directly. They need a stable actor value that preserves causality.

## Choosing Actor or ActorId

Use `ActorId` when the actor can be represented by one canonical identifier. The value may be a prefixed id or resource-name-like string, such as `users/usr_123`, `services/scheduler`, `systems/cron`, or `integrations/github`.

Use `Actor` when consumers need the actor class and local id as separate fields. This is useful when a service wants to keep the actor taxonomy explicit instead of encoding it in one reference value.

For example, generic event metadata can use `ActorId` when the platform standardizes self-describing actor identifiers:

```protobuf
message EventMetadata {
  trogon.actor.v1alpha1.ActorId actor = 1;
  trogon.actor.v1alpha1.ActorId on_behalf_of = 2;
}
```

The protobuf JSON shape is object-based:

```json
{
  "actor": {
    "value": "users/usr_123"
  },
  "onBehalfOf": {
    "value": "services/support-console"
  }
}
```

In that contract, `ActorId` already owns the actor-reference concept. A separate `ActorRef` type would describe the same concept with a different name.

The field inside `ActorId` is named `value` because the message itself carries the domain role. Renaming it to `ref` would make JSON and generated APIs churn without adding a separate concept.

Use `Actor` when the split is part of the contract:

```protobuf
message EventMetadata {
  trogon.actor.v1alpha1.Actor actor = 1;
}
```

```json
{
  "actor": {
    "type": "user",
    "id": "usr_123"
  }
}
```

`type` is an open, stable token instead of an enum because actor taxonomies vary across services and may evolve independently. Use generic values such as `user`, `service`, `system`, `scheduler`, or `integration` when they are precise enough. Use namespaced values when a service owns a more specific actor class.

`id` is an opaque identifier scoped by `type`. It is not display text, and consumers should compare actors by both fields.

An event-specific field can also use `ActorId` when the domain fact is the canonical actor identifier:

```protobuf
message AccountCreated {
  trogon.actor.v1alpha1.ActorId created_by = 1;
}
```

The protobuf JSON shape for `ActorId` is:

```json
{
  "createdBy": {
    "value": "users/usr_123"
  }
}
```

Changing a field from `ActorId` to `Actor` later changes the protobuf and JSON shape for that field. Choose `Actor` up front when consumers need separate actor fields rather than one canonical actor identifier.

## Object ID Values

An actor id may use the same string format as an object id, but the field type should follow the domain role, not the string format.

Use `Actor` or `ActorId` when the field answers who caused the event:

```protobuf
message AccountCreated {
  trogon.actor.v1alpha1.ActorId created_by = 1;
}
```

```json
{
  "createdBy": {
    "value": "users/usr_123"
  }
}
```

Do not use `ActorId` for fields that identify the object being created, updated, or referenced. Those fields should use the domain object's own id type or string field:

```protobuf
message AccountCreated {
  string account_id = 1;
  trogon.actor.v1alpha1.ActorId created_by = 2;
}
```

If the event contract already implies the actor namespace, a shorter id can still be carried by `ActorId`:

```protobuf
message UserProfileUpdated {
  trogon.actor.v1alpha1.ActorId updated_by = 1;
}
```

In that case, the value is still an actor identifier in the event, not a general object reference. Prefer self-describing values in shared event envelopes so replay, projection, and audit consumers do not need external context to interpret the actor.
