# Actor Identity

The `trogon.actor.v1alpha1.Actor` message records who caused a domain event or action when consumers need both an actor type and an actor id.

At application boundaries, the caller may be represented as an authenticated principal, service account, workload identity, or system process. Domain events should not need to expose that IAM vocabulary directly. They need a stable actor value that preserves causality.

## Choosing Actor or ActorId

Prefer `Actor` for shared event metadata and any field where consumers may need to distinguish users, services, systems, schedulers, integrations, or other actor classes. This is the safer default for reusable event envelopes because the actor taxonomy remains part of the field contract.

Use `ActorId` only when the surrounding event contract already fixes the actor class and the string identifier alone is the durable domain fact. Do not use `ActorId` just to avoid the `type` field in JSON.

For example, common event metadata should usually use `Actor`:

```protobuf
message EventMetadata {
  trogon.actor.v1alpha1.Actor actor = 1;
}
```

The protobuf JSON shape is object-based:

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

An event-specific field can use `ActorId` when the domain fact is only the identifier and the actor class is already implied:

```protobuf
message AccountCreated {
  trogon.actor.v1alpha1.ActorId created_by = 1;
}
```

The protobuf JSON shape for `ActorId` is:

```json
{
  "createdBy": {
    "value": "usr_123"
  }
}
```

Changing a field from `ActorId` to `Actor` later changes the protobuf and JSON shape for that field. Choose `Actor` up front when a later type distinction would belong to the same event contract.

## Object ID Values

An actor id may use the same string format as an object id, but the field type should follow the domain role, not the string format.

Use `Actor` or `ActorId` when the field answers who caused the event:

```protobuf
message AccountCreated {
  trogon.actor.v1alpha1.Actor created_by = 1;
}
```

```json
{
  "createdBy": {
    "type": "user",
    "id": "usr_123"
  }
}
```

Do not use `ActorId` for fields that identify the object being created, updated, or referenced. Those fields should use the domain object's own id type or string field:

```protobuf
message AccountCreated {
  string account_id = 1;
  trogon.actor.v1alpha1.Actor created_by = 2;
}
```

If the event contract already implies the actor type, an object-id string can be carried by `ActorId`:

```protobuf
message UserProfileUpdated {
  trogon.actor.v1alpha1.ActorId updated_by = 1;
}
```

In that case, the value is still an actor identifier in the event, not a general object reference.
