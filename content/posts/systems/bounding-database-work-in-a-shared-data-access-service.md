---
title: "Bounding Database Work in a Shared Data Access Service"
date: 2024-07-05T10:00:00+08:00
draft: false
categories: ["Distributed Systems"]
description: "An engineering case study in connection budgets, bounded admission, binary protocol migration, and cache behavior for a shared data access service."
---

An application can run out of database capacity while most of its connections are idle. Every independently scaled worker brings a connection pool, every pool reserves some headroom, and a deployment temporarily runs both generations. The database sees the sum of those decisions. No individual service needs to be unusually busy for the aggregate to become expensive.

Consider a shared data access service used by short-lived workers and long-running applications. It exposes bounded record operations over MySQL and caches reusable reads in Redis. The engineering problem is to absorb changing client concurrency without allowing connection growth, serialization work, and cache misses to amplify one another.

```text
     workers and application services
       bounded calls, explicit deadlines
                      |
          shared data access service
     authentication / operation allowlist
       admission / coalescing / encoding
              /                  \
     Redis read cache       small client pools
                                    |
                             managed SQL proxy
                                    |
                              MySQL primary
                                    |
                         existing change consumer
                                    |
                       invalidate all cache formats
```

This is a worked engineering case study, with generic records and illustrative capacity arithmetic. It describes a service-level redesign and its verification plan, rather than asserting measured production gains. The scope covers connection behavior, the request path, and client migration. Database ownership and the existing change-capture infrastructure remain separate responsibilities.

## 1. Explain why scaling the callers makes the database slower

### 1.1 Separate connection count from useful concurrency

The first investigation should align application and database timelines. A connection spike that overlaps a deployment suggests a different cause from a connection spike driven by slow queries. Request traces need separate spans for admission wait, connection acquisition, SQL execution, cache access, and encoding. Otherwise, a request waiting for a free connection can be misclassified as an expensive query.

A useful connection estimate starts with `replicas × pool limit`, then includes deployment overlap, background jobs, and administrative access. It is an upper bound on application connections, not a prediction of simultaneously executing statements. Connection establishment rate also matters: a moderate steady population can still be disruptive if it reconnects repeatedly during a failure.

The corresponding workload model groups operations by query shape, result size, transaction duration, and cacheability. A bounded point read and an unbounded scan cannot share one meaningful latency distribution. Neither should they receive the same admission policy merely because both use SQL.

I would compare query plans and database waits with application CPU and allocation profiles before selecting a remedy. If one missing index dominates execution time, a proxy adds a hop without removing the expensive operation. If connections multiply during replica growth while query execution stays stable, pooling and admission become more plausible interventions. The design needs evidence for both; it cannot infer the bottleneck from the name of a saturated metric.

### 1.2 Establish a budget that survives deployment overlap

Suppose a load test identifies 120 active short transactions as a conservative operating point for the target workload. This is an illustrative budget, not a database connection setting or a reported result. Reserve 20 for maintenance and other known callers. The service may consume the remaining 100, including replicas being drained during a rollout.

If the deployment permits ten live service replicas during overlap, a local cap of ten active database operations per replica keeps their sum within that allocation. A configuration change that permits twenty replicas invalidates the arithmetic. The maximum replica count and the per-replica cap must therefore be reviewed together. Uneven load can strand capacity on quiet replicas; that is an accepted limitation of a simple static allocation.

This budget covers admitted database operations, not just open sockets. A connection limit still needs its own headroom and enforcement. Other database users must respect their allocations, and long transactions must be counted for their entire lifetime. A statement fan-out cannot acquire one permit and quietly execute ten concurrent queries.

For this service, I prefer a conservative allocation over introducing a distributed permit coordinator immediately. If the resulting utilization is inadequate, a shared allocator becomes a separate design with lease, failure, and overshoot semantics. The current change should first establish a limit that can be explained and tested.

## 2. Put the bound at the point where work becomes expensive

### 2.1 Connection multiplexing solves only part of the problem

A managed SQL proxy can reuse database connections across client sessions, especially for short transactions. It does not turn database execution into an unlimited resource. Session state and certain operations can prevent reuse; the actual driver and query patterns need testing. AWS documents these constraints in its [RDS Proxy pinning guidance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-pinning.html).

The integration should exercise the application's real connection initialization, prepared statements, transaction boundaries, and failure paths. A trivial `SELECT 1` test is insufficient. Observe client connections, database connections, borrow latency, and pinned sessions together. If pinned sessions grow with client count, increasing the client pool can make the original problem reappear behind another endpoint.

Connection acquisition, SQL execution, and cancellation also need independent controls. A caller deadline should not expire while the driver continues using a connection indefinitely. Where cancellation cannot establish that the connection is clean, discard it according to the driver's contract rather than returning potentially active transactional state to the pool.

This creates an important operational distinction: the proxy owns reusable database connections; the application owns how much work it attempts. Neither layer can substitute for the other's limit.

### 2.2 Use bounded queues and carry the remaining deadline

The service admits a request only when it can plausibly finish within the caller's remaining budget. It also bounds the number and total estimated bytes of waiting requests. A request-count limit alone offers little protection when one message can contain a much larger record than another.

The request path can be expressed as the following pseudocode:

```text
validate identity, operation and payload bounds
compute remaining request budget
try to enter bounded admission queue
    reject if full or already too late
acquire operation-class permit before queue deadline
    remove request if cancelled while waiting
check cancellation again
execute bounded database operation
    propagate remaining time to supported driver controls
release permit only after database work has stopped
encode response within output-size limit
```

The permit covers actual downstream occupancy. Releasing it merely because the RPC handler returned would allow cancelled queries to accumulate invisibly. This distinction should appear in the integration test and the operational dashboard.

The gRPC transport supports deadlines, but application code must still stop unnecessary work and account for ambiguous outcomes. The original [gRPC deadline guidance](https://grpc.io/blog/deadlines/) explains that the server can finish successfully even when the client times out. Consequently, automatic retries are limited to explicitly safe reads. A mutation needs its own deduplication or reconciliation contract before an uncertain result can be retried.

Fairness is another practical constraint. Separate bounded lanes for interactive requests and batch work keep a large import from consuming every waiting slot. A small per-caller cap prevents one client from occupying the entire interactive lane. These are service policies with configurable limits, rather than an attempt to build a universal scheduling platform.

### 2.3 Make overload visible to clients

An overload response should identify a retryable capacity rejection without exposing internal SQL details. Clients use bounded attempts, jitter, and a shared end-to-end deadline. An immediate retry from every worker simply moves the queue into the network.

Timeouts, explicit rejection, and dependency failures remain separate metrics. A lower successful-request P99 is not progress if the service rejected a larger fraction of calls. The performance report must include offered load, successful throughput, rejection rate, and timeout rate together.

During a proxy outage, unrestricted direct database access is particularly dangerous: it removes the control at the moment callers are most likely to retry. An emergency route would need its own tested, reserved budget. Without that preparation, bounded rejection is a more defensible behavior than a fleet-wide bypass switch.

## 3. Reduce serialization cost without changing the operation contract

### 3.1 Define a small API instead of forwarding arbitrary SQL

The shared service earns its extra network hop only if it provides useful policy: bounded operations, authorization, predictable response shapes, and common telemetry. If connection pooling were the only requirement, clients connecting directly through the managed proxy would be a simpler alternative.

The binary API therefore describes record operations rather than accepting SQL strings. For example, a read request carries an object identifier and an explicit consistency choice. The server maps that operation to a reviewed query. Identity comes from authenticated context, and authorization is applied before cache lookup. A cache key incorporates the isolation boundary, object identity, and any response variant that changes visible data.

```proto
syntax = "proto3";
package access.v2;

message ReadRecordRequest {
  string object_id = 1;
  enum ReadMode {
    READ_MODE_UNSPECIFIED = 0;
    ALLOW_CACHED = 1;
    PRIMARY = 2;
  }
  ReadMode mode = 2;
  reserved 3; // Retired field; never reuse its number.
}

message ReadRecordResponse {
  string object_id = 1;
  uint64 revision = 2;
  bytes payload = 3;
}
```

The unspecified mode is rejected, and unknown enum values receive a defined error. An opaque payload is appropriate only if its encoding and size are separately specified; stable, frequently accessed fields should have typed definitions. Replacing an arbitrary JSON blob with arbitrary bytes would otherwise preserve much of the decoding cost while making debugging harder.

Protocol Buffers field numbers identify fields on the wire. Deleted numbers must not be reused, and wire compatibility does not guarantee equivalent application behavior. These rules come from the [proto3 language guide](https://protobuf.dev/programming-guides/proto3/). Compatibility fixtures should test the generated clients actually deployed, including older readers receiving a newer response.

### 3.2 Attribute the benefit to the right change

The experiment compares the old request path, the same operation behind admission controls, and the binary path. Encoding changes should not be credited with gains caused by a new index or a different query projection. Where a full matrix is too expensive, fixed query fixtures isolate serialization before an integrated load test measures the complete path.

Measure CPU per completed operation, allocated bytes, response bytes, and latency across the payload-size distribution. Protobuf may avoid repeated field names and some textual conversions, but it is not a guarantee of smaller application objects or lower resident memory. Generated messages, temporary buffers, and cache retention still consume memory.

Compression is evaluated separately. Small messages may not justify its CPU cost, while large compressible payloads may benefit. The decision should follow the actual workload and resource budget rather than a blanket switch applied to every response.

Operational tooling belongs in the migration as well. A schema-aware inspection command can decode an authorized sample without enabling routine payload logging. Production traces retain operation names, sizes, timings, and safe identifiers. Making binary data debuggable does not require copying private records into logs.

## 4. Treat the cache format change as a correctness change

### 4.1 Versioned keys prevent decoding errors, not stale reads

During a rolling deployment, old code expects JSON while new code expects a binary representation. Separate namespaces, such as `record:v1:<scope>:<id>` and `record:v2:<scope>:<id>`, keep those formats distinguishable. Both include the same authorization scope.

That separation introduces a new obligation: every invalidation path must cover both namespaces before new readers are enabled. Invalidating only the format written by the current process leaves the other generation serving stale records. An old server participating in the rollout is still a writer from the cache's perspective.

The least complicated migration here avoids promoting an old cached value into the new namespace. On a v2 miss, the new service reads the database under its normal admission budget and populates v2. This increases cold-cache database demand, so the rollout fraction and refill concurrency are deliberately bounded. It avoids creating another asynchronous conversion path whose stale writes must be fenced.

Rollback is similarly explicit. First disable new fills, then bypass both caches while old in-flight fills finish or expire under a bounded request lifetime. Rotate to a fresh namespace before resuming cached reads. A format switch alone cannot promise a safe return to an old namespace that may contain outdated data.

### 4.2 State the freshness contract honestly

Cache-aside plus asynchronous invalidation has a familiar race: a reader obtains revision 7, a writer commits revision 8 and invalidates, and the reader then fills revision 7. Merely deleting the key after a write does not prevent this ordering.

For the scope of this service, cached reads explicitly allow staleness. Fills use an expiration timestamp derived from when the authoritative read began, not a fresh full TTL after an arbitrarily delayed callback. A fill arriving after that deadline is discarded. Both key formats use the same policy, fills read the primary, and stale entries are never copied between namespaces with a renewed lifetime. This limits how long a pre-update read can remain eligible for caching under the assumed bounded request lifetime and clock policy; invalidation improves freshness but is not the sole bound.

Operations requiring current state bypass Redis and read the primary. A caller's immediate read after a mutation uses this path. Routing that request to a lagging replica would undermine the contract even though the cache was bypassed.

A stronger cached guarantee would require a durable revision or invalidation-generation protocol shared by all readers and writers. That is a legitimate extension, but the initial performance change does not need to imply it. The service can offer a narrow strong path alongside a clearly described stale-tolerant one.

### 4.3 Keep coalescing and cache failure within the same budget

Concurrent misses for one key can share a local in-flight read. The leader performs the database operation; followers wait only until their own deadlines. The in-flight map has a size limit and removes entries on success, failure, and cancellation. A follower timing out does not cancel a leader still serving other callers.

Local coalescing does not eliminate duplicate work across replicas. The database admission budget remains the final bound. During Redis failure, cache misses must still pass through that budget; a fallback path that bypasses it converts a cache outage into a database outage.

Cache outages also exercise response semantics. Stale data can be returned only for operations whose contract permits it. The service must not synthesize an apparently valid record for an authoritative read simply to keep an availability graph green.

## 5. Deliver a service change that other teams can adopt

### 5.1 Separate the rollout into reversible decisions

The implementation sequence begins with instrumentation and operation bounds on the existing protocol. Connection-proxy integration follows with fixed pool sizes and representative pinning tests. Binary RPC comes next for a small set of read operations, followed by cache-format migration. This ordering makes regressions attributable and gives each stage an independent stop condition.

Read-only shadow comparisons can check semantic equivalence, but their extra database work must be budgeted. Mutations are never duplicated merely to compare implementations. For reads, compare normalized fields and errors, not serialized byte equality. Different encoders can represent equivalent data differently.

A stable client cohort receives the new path first. Expansion depends on error parity, bounded resource use, and latency at the same offered load. Rollback preserves the admission limits; reverting a transport should not reopen unrestricted database concurrency.

### 5.2 Define the evidence before claiming an improvement

The validation plan includes steady traffic, deployment overlap, reconnect bursts, a cold cache, and prolonged downstream slowness. Each run records client attempts as well as server completions so that retries and dropped requests cannot disappear from the report.

| Experiment | Evidence required |
| --- | --- |
| Increase callers at fixed service budget | Database occupancy remains bounded; excess work is rejected predictably |
| Add a long-lived or pinned workload | Borrow latency and pinned-session signals identify lost reuse |
| Remove Redis | Database admission remains effective; memory does not grow with waiting calls |
| Mix old and new clients | Equivalent read semantics, supported schemas, invalidation of both formats |
| Expire calls during database work | Permits remain held until work stops; no connection returns with an open transaction |
| Roll back after v2 has served traffic | Old decoding works and stale namespaces are not silently reactivated |

The handoff consists of a client migration guide, the tested driver configuration, dashboards, and a runbook describing which control to adjust for which symptom. Application teams own adoption and operation-specific semantics; the change-consumer owner verifies invalidation coverage. The service engineer owns the request-path implementation and the evidence that its bounds survive those integrations.

There are deliberate costs. Static per-replica budgets sacrifice some utilization. A shared service adds a hop and another deployment. Strong reads consume database capacity even when Redis is healthy. Two cache formats temporarily require more memory. Those costs are justified only if the resulting service behaves predictably under the very conditions that made the original path unstable.

The next optimization should follow the new limiting resource. If SQL execution dominates after connections and encoding are controlled, query design is the next project. If the service's own CPU saturates first, profiles should identify whether copying, validation, or dispatch now deserves attention. A successful performance change leaves that next decision easier to make.
