# Hierarchy Node

The `trogon.hierarchy.v1alpha1.NodeId` message identifies one node of a tenant's hierarchy: the position a resource attaches to, the anchor policy attaches to, and the point name resolution starts from.

A node is untyped. Tenants model their own structure, so one tenant nests teams inside departments while another nests client accounts, and the platform never conditions behavior on what a node is called. Words such as "project" or "team" are labels on a node, not kinds.

## Using NodeId

A resource with a position in the tree carries it in a field named `parent`:

```protobuf
message Agent {
  string agent_id = 1;
  trogon.hierarchy.v1alpha1.NodeId parent = 2;
}
```

```json
{
  "agentId": "agt_123",
  "parent": {
    "value": "node_01h9x"
  }
}
```

The field name states the role and the type states what the role points at. A resource may sit at a node of any kind, so `parent` is a role rather than a type, which is why the field is not named after one. Spelling it `parent_id` repeats the id suffix that already lives in the type.

The field inside `NodeId` is named `value` for the same reason it is in `ActorId`: the message carries the domain role, so the field does not need to restate it.

## One Node, Not a Path

The value identifies the immediate node and nothing else. It is `node_01h9x`, never `acme/backend/ml`.

Encoding structure in the string looks convenient and fails on the first reorganization: a rename or a move invalidates every stored pointer, and history that is immutable cannot be repaired. Keeping the value opaque means the tree changes through audited add, move, and remove operations while identifiers stay put, so a move rewrites nothing.

The consequence is that ancestry is a query. Walking from a node toward the root, whether to resolve a name, evaluate inherited policy, or find who is accountable, asks the hierarchy rather than parsing a string. That is the trade that makes a move touch one record instead of every descendant.

## Kinship Is Not Placement

A tree of same-kind resources, a session spawned by a session or an agent delegating to an agent, is kinship, not a position in the tenant's hierarchy. Kinship uses the other resource's own id type in a qualified field:

```protobuf
message Session {
  string session_id = 1;
  trogon.hierarchy.v1alpha1.NodeId parent = 2;
  SessionId parent_session = 3;
}
```

A bare `parent` always means the hierarchy. A kinship field always names the type it points at.

## Ownership Is Not a Separate Field

Attaching a resource to a node is an authorized write: the authorization system decides whether the caller may attach there, and whoever answers for the node answers for what hangs off it. Resources therefore do not carry an owner field by default. Copying the answer onto each resource creates a second record of one fact, and the copy is the one that goes stale when the tree is reorganized.

Resolution walks up: the nearest ancestor with an accountable principal wins, and the tenant root always has one, so every resource resolves to exactly one answer.
