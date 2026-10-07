# Consistency Pattern

## Overview

The `trogon.consistency.v1alpha1` package provides Protobuf message definitions for expressing consistency requirements on queries in event sourced systems. It lets a client state, per query, how the response should relate to the position of the event log: accept whatever a projection currently reflects, wait for a specific position, wait for an exact snapshot, or wait for the head of the log as observed at request time.

## Problem Statement

In an event sourced system, there is a delay between an event being committed to the event log and a read model finishing processing it. This creates a race condition: a client performs a mutation, then immediately queries, and gets a response that does not yet reflect that mutation.

Different queries tolerate this race differently. A dashboard does not care. A confirmation page shown right after a mutation does. An authorization check before a destructive action cannot tolerate it at all. The `Consistency` message lets each query state which of those situations it is in.

## Versions in event sourcing

An event sourced system exposes distinct counters, and the `Consistency` message is only meaningful once the difference between them is clear.

The **stream version** is the position within one aggregate's event stream. It increases monotonically for that stream only. The **global position** is the position within the all-events log. It increases monotonically across every stream in the system.

A **projection checkpoint** is the position a read model has processed up to. A projection built from a single stream (for example, a projection that only ever reads one aggregate's events) checkpoints on stream version. A projection built from many streams (for example, a list view, a search index, or any cross-aggregate read model) checkpoints on global position.

This gives a rule that every service using `Consistency` must follow: the `version` a client passes in `min_version` or `exact_version` must be the same counter the target projection checkpoints on. A service must document, per query, which counter it expects. The mutation response that hands the client a version must return that same counter, not the other one. Passing a stream version to a query whose projection checkpoints on global position (or the reverse) produces a request that can never be satisfied, or one that is satisfied by the wrong events.

### Where the version comes from

The write side has to hand the client the counter the target projection checkpoints on, and how that counter becomes available depends on what the event store returns when a stream is appended to.

Some stores return every counter on append. EventStoreDB's append result carries the stream revision of the last appended event and its position in the `$all` log, so a mutation handler can return either without a second read. Each event delivered by a subscription carries the same pair, and a projection records whichever one it consumes: a catch-up subscription to `$all` checkpoints on the log position, a subscription to a single stream checkpoints on the stream revision. When the store exposes the log position as a composite (EventStoreDB's commit and prepare positions), the commit position is the usual choice for the `int64` value, with the checkpoint recorded once every event sharing that commit position has been processed.

Some stores return only the stream version on append. Commanded's dispatch result exposes the aggregate version, while the global event number appears only on the recorded events later delivered to event handlers and projectors. Commanded's Ecto projections checkpoint on that global event number. A mutation handler that needs it reads the appended events back from the aggregate's stream and takes the global number of the last one, which is one extra read per mutation taken on the write path.

The translation can instead happen at query time. The mutation returns the stream version, and the query handler locates that event in its stream to learn the global position before comparing with the checkpoint. This keeps the write path free of extra reads at the cost of one read per consistent query, and it lets the API contract speak only in stream versions, which is simpler for clients.

Principles that hold regardless of store:

- Whatever counter crosses the API boundary, the server must be able to compare it with the projection checkpoint without guessing. Document which counter each mutation returns and which counter each query expects.
- Prefer returning the counter the projection checkpoints on directly when the store provides it on append.
- When the store provides only the stream version, choose one translation point per service, write path or query path, and keep it consistent.
- The counter is a log position, not a clock. Never derive it from timestamps, and never compare counters from different stores or different event logs.

Some frameworks also offer a write-side alternative. Commanded can block a dispatch until strongly consistent handlers have processed the new events, which removes projection lag for the caller at the cost of slower writes. This package expresses the same need on the read side instead, so that each query pays only for the consistency it asks for. A service may use either, but should not wait on both sides for the same projection.

The `version` field is `int64`, deliberately. It is not an opaque token; it is the actual event sourcing position, and a client is free to compare two versions or reason about their ordering. This is unlike SpiceDB's `ZedToken`, which is opaque by design and only usable as an input to a later request, never inspected or compared by the client.

## Modes

### minimize_latency

Also the behavior when `requirement` is left unset entirely.

- **Guarantee**: the response reflects some consistent prefix of the event log. No freshness bound is promised. The prefix is always a single position; a response never mixes data from two different points in the log.
- **Input**: none.
- **Waits**: never. `timeout_duration` and `delay_duration` are ignored.
- **Cache allowed**: yes, and expected. A server may intentionally serve a slightly older position so that concurrent callers share one cached result instead of each recomputing it.
- **Output**: the result. The server should echo the position it served from, so the client can later issue a `min_version` request against that same position if it needs to confirm freshness.
- **Failure**: none specific to this mode. It cannot time out and nothing it returns can expire.
- **When to use**: browsing, listing, dashboards, and any read where staleness is acceptable.

Setting `minimize_latency` explicitly behaves exactly like leaving `requirement` unset. The explicit form exists so the intent is visible in code review, in logs, and in traces, rather than looking like a caller who forgot to set a mode.

### min_version (read-your-writes)

- **Guarantee**: the response reflects every event up to and including `version`, and possibly later events too.
- **Input**: `version`, taken from a previous mutation response.
- **Waits**: yes, until a result reflecting a position greater than or equal to `version` is available, bounded by `timeout_duration` and retried every `delay_duration`.
- **Cache allowed**: only if the cached result's position is greater than or equal to `version`.
- **Output**: the result.
- **Failure**: `UNAVAILABLE` if `timeout_duration` elapses before the projection catches up.
- **When to use**: immediately after a mutation performed by the same client, to read back what was just written. This is the common mode.

### exact_version (point-in-time snapshot)

- **Guarantee**: the response reflects exactly the events up to `version`, and nothing after it.
- **Input**: `version`.
- **Waits**: yes, until a result reflecting exactly `version` is available, bounded and retried the same way as `min_version`.
- **Cache allowed**: only if the cached result is for exactly that position, not a later one.
- **Output**: the result.
- **Failure**: `UNAVAILABLE` if `timeout_duration` elapses. `FAILED_PRECONDITION` if the projection has already moved past `version` and the server cannot reconstruct state as of that position (the snapshot has expired).
- **When to use**: multiple reads that must agree with each other, such as paginating a report, computing a diff between two calls, or an audit. Not a tool for freshness. Passing the latest known version to `exact_version` to mean "give me the newest data" is a mistake: the request fails the moment any other write lands, because the projection has then moved past the requested version.

### fully_consistent (head of log)

- **Guarantee**: the response reflects every event committed to the event log before the request arrived at the server. The server, not the client, determines that head position by observing the log at the moment the request arrives.
- **Input**: none. A client that holds a version should use `min_version` instead; `fully_consistent` is for a client that has no version to offer.
- **Waits**: yes, until a result reflecting a position at or past the head observed at request arrival is available, bounded by `timeout_duration` and retried every `delay_duration`. Alternatively, when a query targets a single stream, the server may answer directly from that stream's current state instead of waiting on a projection.
- **Cache allowed**: no. Any result that may predate the request's arrival at the server must not be served.
- **Output**: the result. The server should echo the head position it satisfied, so the caller can downgrade to `min_version` on a follow-up read instead of paying for `fully_consistent` again.
- **Failure**: `UNAVAILABLE` if `timeout_duration` elapses. Servers may clamp `timeout_duration` tighter for this mode, and may rate limit it, because it is the most expensive mode to satisfy.
- **When to use**: decisions where any staleness is unacceptable and the caller holds no version to pin to, such as an authorization check immediately before a destructive action, or a balance check immediately before a withdrawal. This mode should be rare.

## Comparison

| | minimize_latency | min_version | exact_version | fully_consistent |
|---|---|---|---|---|
| Input version | none | required | required | none |
| Waits | never | yes, bounded | yes, bounded | yes, bounded |
| Newer data than target accepted | n/a | yes | no (fails) | n/a, target is observed at request time |
| Cache allowed | yes, expected | only if position >= version | only if position == version | no |
| Failure | none | UNAVAILABLE on timeout | UNAVAILABLE on timeout, FAILED_PRECONDITION on expired snapshot | UNAVAILABLE on timeout |
| Relative cost | lowest | low | low | highest |
| Typical use | browsing, dashboards | read-your-writes after a mutation | reports, audits, repeatable reads | pre-destructive-action checks |

## Usage Pattern

### 1. Mutation Returns Version

```protobuf
// Example mutation response
message CreateOrderResponse {
  string order_id = 1;
  int64 version = 2;  // The counter the target projection checkpoints on
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

1. Perform a mutation and capture the version from the response (for example, version 5), using the same counter the target query's projection checkpoints on.
2. Immediately query with a consistency requirement:
   - Set `min_version.version` to the captured version for read-your-writes.
   - Set `exact_version.version` for a repeatable, point-in-time read.
   - Set `fully_consistent {}` when no version is held and staleness cannot be tolerated.
   - Leave `requirement` unset, or set `minimize_latency {}`, when staleness is acceptable.
   - Configure `timeout_duration` (for example, 1s) and `delay_duration` (for example, 100ms) for any waiting mode.
3. Server retries until the required position is reached or the timeout elapses.
4. Client receives a result that satisfies the requested guarantee.

## Server Implementation Guidelines

### Recommended Limits

Servers should enforce reasonable limits on the waiting modes (`min_version`, `exact_version`, `fully_consistent`):

- **timeout_duration**:
  - Default: 1s (if not provided)
  - Maximum: 5s (clamp values above, and consider a tighter maximum for `fully_consistent`)

- **delay_duration**:
  - Default: 100ms (if not provided)
  - Maximum: 500ms (clamp values above)

### Error Handling

When a projection fails to meet a consistency requirement, return the appropriate gRPC status code:

**Projection Timeout** (`min_version`, `exact_version`, or `fully_consistent`, timeout exceeded):
- Status Code: `UNAVAILABLE` (503)
- Use `google.rpc.ErrorInfo` for structured error details
- Suggested metadata: `requiredVersion` (or `requiredHeadPosition` for `fully_consistent`), `currentVersion`, `attempts`, `elapsedMs`

**Snapshot Expired** (`exact_version` only, projection moved past the requested version):
- Status Code: `FAILED_PRECONDITION` (400)
- Use `google.rpc.ErrorInfo` for structured error details
- Suggested metadata: `requestedVersion`, `currentVersion`

Consider using `google.rpc.Status` with `google.rpc.ErrorInfo` for rich error responses that clients can programmatically handle.

### Retry Logic

For the waiting modes:

```
1. Determine required position:
   - min_version: required = requested version
   - exact_version: required = requested version
   - fully_consistent: observe log head H at request arrival; required = H
2. Execute the query and capture the position the result reflects.
   The result and its position must come from the same projection snapshot
   (for example, the same read transaction, or the same cached entry).
3. Compare that result position, not a later checkpoint, with required.
4. If min_version or fully_consistent and result position >= required -> return result
5. If exact_version and result position == required -> return result
6. If exact_version and result position > required -> return FAILED_PRECONDITION
7. If timeout exceeded -> return UNAVAILABLE
8. Sleep for delay_duration
9. Goto step 2
```

The position must be read together with the result. Reading the projection checkpoint after the query is a race: the projection can advance between the query and the checkpoint read, so a checkpoint that satisfies `required` says nothing about the rows that were returned. Under `exact_version` the same race can return rows from before the requested position while the checkpoint reads as equal to it. Every projection that supports a waiting mode therefore needs a way to return its position atomically with the query result.

`minimize_latency` skips this loop entirely: it executes the query once against whatever the projection currently reflects and returns immediately.

### Special Case: NOT_FOUND Errors

Under `min_version`, `exact_version`, or `fully_consistent`, a `NOT_FOUND` result carries a position like any other result, and the same comparison applies to it:

- If the result position is below `required`, the entity may not exist yet because of projection lag. Treat it as not satisfied and retry until timeout.
- If the result position satisfies `required`, the entity genuinely does not exist at that position. Return `NOT_FOUND`. Retrying would only convert a correct answer into `UNAVAILABLE` after the timeout elapses.

## Observability

### Metrics

Track these metrics:

- `consistency.attempts` - Distribution of retry attempts
- `consistency.latency` - Time to satisfy the consistency requirement
- `consistency.timeouts` - Count of timeouts
- `consistency.snapshot_expired` - Count of `exact_version` failures

### Logging

Log when:
- Clamping client-provided timeout/delay values
- Timeout exceeded
- Snapshot expired (`exact_version`)
- Retry attempts (at debug level)

### Tracing

Add span attributes:
- `consistency.mode` - one of `minimize_latency`, `min_version`, `exact_version`, `fully_consistent`
- `consistency.required_version` - requested version, for `min_version` and `exact_version`
- `consistency.observed_head` - head position observed at request arrival, for `fully_consistent`
- `consistency.actual_version` - projection version when satisfied
- `consistency.attempts` - number of retries
- `consistency.elapsed_ms` - time taken

## Trade-offs

### Pros
- Lets each query state its own freshness requirement instead of a single global setting
- No changes to the event store required
- Client controls the consistency versus latency trade-off per request
- Works with any projection technology

### Cons
- The waiting modes increase query latency through retry loops
- The waiting modes increase server load through repeated queries
- `min_version` and `exact_version` require the client to track and pass versions
- `fully_consistent` is the most expensive mode and needs its own limits
- Does not work with projections that do not track a position

## When to Use

**minimize_latency**: browsing, listing, dashboards, anywhere staleness is acceptable.

**min_version**: right after a mutation, to read back what was just written.

**exact_version**: repeatable reads that must agree with each other, such as paginated reports or audits. Not for freshness.

**fully_consistent**: decisions where no staleness is acceptable and no version is held, such as an authorization check before a destructive action. Use sparingly.

**Avoid any waiting mode for**: high-throughput, read-heavy workloads where the added latency and server load outweigh the freshness benefit.

## Reference

Inspired by:
- **SpiceDB**: `minimize_latency`, `at_least_as_fresh`, `at_exact_snapshot`, and `fully_consistent` modes
  - https://authzed.com/docs/spicedb/concepts/consistency

- **DynamoDB**: Consistent reads
  - https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

- **Cassandra**: QUORUM/ALL consistency levels
  - https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html#consistency-levels
