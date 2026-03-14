# Consistency Pattern

## Overview

The `trogon.consistency.v1alpha1` package provides Protobuf message definitions for implementing **read-after-write consistency** in eventual consistency systems, particularly CQRS/Event Sourcing architectures.

## Problem Statement

In CQRS/Event Sourcing systems, there's often a delay (projection lag) between:
1. Writing an event to the event store
2. The read model/projection processing that event

This creates a race condition where a client performs a mutation but immediately queries and gets stale data (doesn't see their own write).

## Solution

The `Consistency` message allows clients to specify version requirements for queries, enabling the server to retry until the projection catches up.

## Message Definitions

### Consistency

Main message with two modes:

```protobuf
message Consistency {
  oneof requirement {
    MinVersion min_version = 1;
    ExactVersion exact_version = 2;
  }
  
  optional google.protobuf.Duration timeout_duration = 3;
  optional google.protobuf.Duration delay_duration = 4;
}
```

### MinVersion (Recommended)

Wait for projection to reach **at least** the specified version (`version >= X`).

**Use for**: Read-after-write consistency (most common use case)

```protobuf
message MinVersion {
  int64 version = 1;
}
```

### ExactVersion (Strict)

Wait for projection to reach **exactly** the specified version (`version == X`).

**Use for**: Reproducible queries, audits, point-in-time reports

```protobuf
message ExactVersion {
  int64 version = 1;
}
```

## Usage Pattern

### 1. Mutation Returns Version

```protobuf
// Example mutation response
message CreateOrderResponse {
  string order_id = 1;
  uint64 stream_version = 2;  // ← Returns current event stream version
}
```

### 2. Query with Consistency

```protobuf
// Example query request
message GetOrderRequest {
  string order_id = 1;
  trogon.consistency.v1alpha1.Consistency consistency = 2;
}
```

### 3. Client Flow

1. Perform mutation and capture `stream_version` from response (e.g., version 5)
2. Immediately query with consistency requirement:
   - Set `min_version.version` to the captured version
   - Configure `timeout_duration` (e.g., 1s)
   - Configure `delay_duration` (e.g., 100ms)
3. Server retries until projection catches up or timeout
4. Client receives result guaranteed to include their write

## Consistency Modes Comparison

| Feature | MinVersion | ExactVersion |
|---------|-----------|--------------|
| **Read-Your-Writes** | Yes | Yes |
| **Returns Newer Data** | Yes | Fails (FAILED_PRECONDITION) |
| **Use Case** | Normal queries | Audits, reports |
| **Flexibility** | High | Low |
| **Common** | 99% of cases | 1% of cases |

## Server Implementation Guidelines

### Recommended Limits

Servers should enforce reasonable limits:

- **timeout_duration**:
  - Default: 1s (if not provided)
  - Maximum: 5s (clamp values above)
  
- **delay_duration**:
  - Default: 100ms (if not provided)
  - Maximum: 500ms (clamp values above)

### Error Handling

When projection fails to meet consistency requirements, return appropriate gRPC status codes:

**Projection Timeout** (timeout exceeded):
- Status Code: `UNAVAILABLE` (503)
- Use `google.rpc.ErrorInfo` for structured error details
- Suggested metadata: min_version, current_version, attempts, elapsed_ms

**Snapshot Expired** (ExactVersion only, version moved past):
- Status Code: `FAILED_PRECONDITION` (400)
- Use `google.rpc.ErrorInfo` for structured error details
- Suggested metadata: requested_version, current_version

Consider using `google.rpc.Status` with `google.rpc.ErrorInfo` for rich error responses that clients can programmatically handle.

### Retry Logic

```
1. Execute query
2. Check projection version
3. If version requirement met → return result
4. If ExactVersion and version > required → return FAILED_PRECONDITION
5. If timeout exceeded → return UNAVAILABLE
6. Sleep for delay_duration
7. Goto step 1
```

### Special Case: NOT_FOUND Errors

If query returns NOT_FOUND (entity doesn't exist in projection yet), treat it as projection lag and retry until timeout. The entity might appear once projection catches up.

## Observability

### Metrics

Track these metrics:

- `consistency.attempts` - Distribution of retry attempts
- `consistency.latency` - Time to satisfy consistency requirement
- `consistency.timeouts` - Count of timeouts
- `consistency.snapshot_expired` - Count of ExactVersion failures

### Logging

Log when:
- Clamping client-provided timeout/delay values
- Timeout exceeded
- Snapshot expired (ExactVersion)
- Retry attempts (at debug level)

### Tracing

Add span attributes:
- `consistency.mode` - "min_version" or "exact_version"
- `consistency.required_version` - Requested version
- `consistency.actual_version` - Projection version when satisfied
- `consistency.attempts` - Number of retries
- `consistency.elapsed_ms` - Time taken

## Trade-offs

### Pros
- Enables read-after-write consistency
- No changes to event store required
- Client controls consistency vs latency trade-off
- Works with any projection technology

### Cons
- Increases query latency (retry loops)
- Higher server load (repeated queries)
- Client must track and pass versions
- Doesn't work with projections that don't track version  

## When to Use

**Use Consistency for:**
- Mutations followed by immediate reads (e.g., "Create order → View order")
- User flows requiring strong consistency UX
- Testing/demos where eventual consistency is confusing

**Don't use Consistency for:**
- Browsing/listing queries (eventual consistency is fine)
- Public-facing queries (projection lag usually acceptable)
- High-throughput read-heavy workloads (adds latency)

## Reference

Inspired by:
- **SpiceDB**: `at_least_as_fresh` and `at_exact_snapshot` modes
  - https://authzed.com/docs/spicedb/concepts/consistency
  
- **DynamoDB**: Consistent reads
  - https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html
  
- **Cassandra**: QUORUM/ALL consistency levels
  - https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html#consistency-levels
