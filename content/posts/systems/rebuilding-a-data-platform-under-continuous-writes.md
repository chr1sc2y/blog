---
title: "Rebuilding a Data Platform Under Continuous Writes"
date: 2025-11-14T10:00:00+08:00
draft: false
categories: ["Distributed Systems"]
description: "A worked engineering case study in stable data identity, versioned replication, bounded caching, safe ownership transfer, and measurable recovery."
---

A database migration becomes a distributed-systems problem as soon as the application must keep accepting changes. A bulk copy can move records, but it cannot determine which writer is authoritative, whether a delayed event is still valid, or what happens when a successful write loses its response. Those questions define the project more precisely than the choice of destination database.

Consider a multi-tenant service whose read path assembles frequently requested objects from several document collections. Background imports compete with interactive requests, and a small engineering team operates the storage stack. The redesign separates authoritative records from query projections, moves the primary access path from MongoDB to DynamoDB, and uses Redis selectively for reusable computation.

```text
                           Application boundary
                    authentication, admission, deadlines
                                      |
                             Authoritative writer
                          one active owner per phase
                           /                     \
                MongoDB: initial owner      DynamoDB: target owner
                           |                     |
                 change capture             change capture
                           |                     |
                 durable event journal / projection workers
                                      |
                       query views and Redis aggregates
                                      |
                   version, lag, cost and recovery telemetry
```

This is a worked design case study. The architecture and validation procedures describe an engineered solution, rather than a claim of measured production results. The examples use generic records and infrastructure capabilities available by November 2025.

## 1. Diagnose the service before replacing the database

### 1.1 Establish the expensive unit of work

The initial symptom is rising tail latency during write bursts. The tempting explanation is insufficient database capacity. A more useful investigation asks what work each request creates and where requests wait.

The read path repeatedly performs joins, converts related documents into application objects, and serializes nearly identical responses. The write path admits background imports at a rate determined by incoming work rather than downstream capacity. A storage replacement could leave both problems intact.

The investigation therefore separates database execution, connection acquisition, application CPU, allocation, serialization, and queueing. Traces identify representative request classes; query plans explain storage work; profiles reveal whether time is spent computing or waiting. Background and interactive traffic receive separate labels so that a fast aggregate average cannot conceal an overloaded foreground path.

The output is a workload model: request mix, payload-size distribution, update frequency, tenant concentration, and the most frequently accessed records. Record count alone is a poor capacity estimate. A small collection with one heavily updated object can be harder to serve than a larger collection with uniformly distributed access.

### 1.2 Turn product behavior into storage contracts

The design starts with three explicit contracts:

| Operation | Required behavior | Permitted compromise |
| --- | --- | --- |
| Read one authoritative object | Return committed state; a caller may request at least a known revision | Fail within a deadline if the required state cannot be obtained |
| Browse a query projection | Return a documented ordering and continuation token | Projection lag within the service's freshness objective |
| Execute a mutation | Apply one accepted logical operation once within the deduplication contract | Retryable failure or a pending result when the outcome is unresolved |

The consistency unit is one bounded object. Cross-object workflows need a separate transaction or explicit state machine; the service does not imply atomicity across arbitrary collections. This boundary prevents a migration from accidentally promising semantics that neither the old application nor the new schema implements.

The architecture decision compares tuning the existing document deployment, adopting a managed relational service, and using a managed key-value store. MongoDB remains a viable option if query and deployment improvements remove the bottleneck. DynamoDB becomes attractive when the dominant access paths can be represented by explicit keys and the team values managed capacity and recovery over flexible server-side joins.

My criterion is the cost of operating the complete service. Moving to a managed database while adding several independently operated caches and queues can increase that cost. The decision record needs an explanation of the access patterns gained, the queries made harder, and the conditions that would invalidate the choice.

## 2. Separate identity, distribution, and query shape

### 2.1 Give each record a stable identity

A record's primary identity should survive changes to its grouping, category, or display order. Otherwise, editing a descriptive attribute becomes a delete-and-create operation with consequences for references, caches, replication, and retries.

An illustrative authoritative record has this shape:

```text
PK = TENANT#<tenant_id>#OBJECT#<object_id>
SK = STATE

owner_epoch
revision
schema_version
payload
payload_digest
deleted
```

The identifier is immutable, and tenant identity is part of the access boundary. Application authorization verifies that the authenticated tenant owns the requested key. Constructing a key from an untrusted tenant parameter is insufficient isolation.

The payload stays within a deliberately enforced item-size budget. Large immutable content belongs outside the record, referenced through an immutable content identifier; metadata references are published only after the content is durable. Unreferenced content is reclaimed separately. That avoids turning one oversized document into an unbounded database item or exposing a pointer to an unfinished upload.

### 2.2 Distribute write load without rebuilding the hotspot in an index

Using a complete object identity as the partition key provides many independently addressable keys. It does not make a single hot object's writes parallel. That limitation remains visible in admission policy and capacity tests.

Tenant-wide listings use a separate access path. A sharded index might use:

```text
GSI_PK = TENANT#<tenant_id>#BUCKET#<hash(object_id) mod N>
GSI_SK = <created_at>#<object_id>
```

The creation timestamp is immutable, and the stable identifier breaks ties in ordering. A reader queries the relevant buckets and merges their sorted results. Its continuation state records enough per-bucket progress to avoid silently skipping items. Under concurrent changes, this is a live listing; a true snapshot export uses a separately defined snapshot procedure.

Write sharding trades concentrated traffic for read fan-out. AWS documents both random and calculated sharding strategies; a calculated suffix allows the application to locate a known object's shard without searching all buckets. [Write-sharding guidance](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html)

The same analysis applies to every GSI. A well-distributed base table can still be constrained by a concentrated index key, and index write throttling can affect base-table writes. [GSI back pressure](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/gsi-throttling.html)

The bucket count and hash function are versioned routing decisions. Changing `N` requires an index transition with backfill, validation, and a defined reader switch. It cannot be deployed as an unrelated constant change. A modest fixed bucket count may be the better choice until measurements justify additional routing machinery.

### 2.3 Make projections replaceable

Query views contain only the fields required by their consumers. A view that falls behind can be rebuilt from authoritative records. A view update includes its input revision and transformation version, making it possible to distinguish stale work from a changed computation.

Top-K summaries and complete listings are different interfaces. Reading a fixed-size summary can have bounded work relative to total collection size. Enumerating all matching records still requires reading and returning those records. A GSI improves the access path; it does not make an arbitrarily large result free.

## 3. Build a migration protocol that survives replay

### 3.1 Prepare the source for a meaningful version contract

Every mutation must pass through an inventoried writer. Batch jobs, repair scripts and administrative tools are included. A migration cannot establish exclusive ownership while an undocumented script retains unrestricted write access.

Before copying data, source mutations atomically update an object's payload and increment its revision in the same document operation. Existing unmodified records have a defined baseline revision. Deletes become versioned tombstones during the migration window, and object identifiers are never reused. MongoDB's single-document atomicity provides the required boundary for this scoped aggregate model. [MongoDB atomicity](https://www.mongodb.com/docs/v8.0/core/write-operations-atomicity/)

Capture also has an infrastructure prerequisite: MongoDB change streams require a replica-set or sharded deployment. A standalone source must first be moved to a supported configuration. Source log and image retention must cover the longest expected snapshot, interruption, and recovery interval. [Change-stream deployment and recovery requirements](https://www.mongodb.com/docs/v8.0/changestreams/)

For update events, the design requires the exact post-image for that change. `fullDocument: "updateLookup"` can observe a later committed version, so its contents cannot safely be labeled with an earlier event's revision. With post-image capture enabled, `fullDocument: "required"` fails if the required image is unavailable. The revision and payload are taken from the same image. [Change-stream image options](https://www.mongodb.com/docs/v8.0/reference/method/db.collection.watch/)

This decision increases capture storage and transport volume. That is an explicit trade-off: full replacement states make target replay simpler than independently ordered field patches.

### 3.2 Separate capture progress from application progress

The capture service writes events to a durable journal before advancing its source checkpoint. Each journal entry contains an event identity, source resume information, object identity, source revision, schema version, and an immutable full-state payload or content reference.

A crash after journal persistence but before checkpoint persistence causes duplicate capture. Advancing the checkpoint first could lose an event. The protocol chooses duplication and makes replay safe.

The journal assigns its own sequence for tracking durable progress. MongoDB resume tokens are treated as opaque source positions. Capture ordering, event identity, and per-object revision are separate concepts; a timestamp does not replace all three. In particular, several changes can share a `clusterTime`. [MongoDB update-event semantics](https://www.mongodb.com/docs/v8.0/reference/change-events/update/)

After durably establishing a capture boundary, the service scans source records by a stable traversal key. Source mutations continue, and both scan output and captured events flow through the same version-aware target applier. A scan does not have to represent one global instant for this per-object convergence protocol: a later full state supersedes an earlier one, while captured tombstones account for objects removed during the scan.

Snapshot completion is tracked independently from stream progress. Source checkpoint, durable journal position, and target-applied position are three different watermarks. Out-of-order worker completion advances the applied watermark only when every earlier journal entry has a terminal, accepted outcome.

### 3.3 Apply full states monotonically

The target accepts a state only when its `(owner_epoch, revision)` is greater than the stored pair. Epoch changes come from the ownership protocol, never from a worker guessing that it is newer.

The acceptance rule can be expressed as a small model:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class State:
    owner_epoch: int
    revision: int
    payload_digest: str
    deleted: bool

def classify(current: State | None, incoming: State) -> str:
    if current is None:
        return "apply"

    old_version = (current.owner_epoch, current.revision)
    new_version = (incoming.owner_epoch, incoming.revision)

    if new_version > old_version:
        return "apply"
    if new_version < old_version:
        return "stale"
    if incoming == current:
        return "duplicate"
    return "conflict"
```

This function describes the protocol; calling it before an unconditional database write would introduce a check-then-write race. The applier instead issues a conditional `PutItem` that atomically admits a missing item or a greater version. On rejection, a strongly consistent read classifies the observed state. Concurrent progress can make an event stale after the failed attempt; an equal version with a different digest or deletion flag is a conflict to investigate. [Conditional replacement with PutItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html)

A higher revision is safe to apply out of order because it carries the complete object state. For partial patches, skipping an earlier label update after applying a later status update could lose the label change. The full-image contract is therefore part of the correctness argument, rather than an incidental serialization choice.

A tombstone participates in the same ordering rule. Replaying an earlier live state cannot resurrect a deleted object. Tombstone removal is gated by the supported replay horizon and a rule rejecting expired replay batches. An operator cannot import an old journal after deleting the state that made its events recognizable as stale.

### 3.4 Reconcile before declaring convergence

An empty queue does not establish data correctness. It can also mean a producer stopped, a transformation discarded records, or a consumer acknowledged work incorrectly.

The validator compares logical representations after deterministic normalization. Missing values, explicit nulls, numeric types, timestamp precision and excluded fields have defined rules. The transformation is versioned and deterministic so that replay does not silently change the meaning of historical events. Its version is pinned for a migration run, and the digest covers schema version, deletion state and canonical payload. Changing that transformation requires a separately validated rebuild namespace.

Validation distinguishes missing records, unexpected records, payload differences, revision differences, tombstone differences and quarantined transformations. Online comparisons inspect both revision and content: unequal revisions can reflect ordinary replication lag, while equal revisions with unequal logical content indicate a stronger defect.

Partitioned counts and digests help locate affected ranges, but a matching aggregate checksum is not the sole correctness proof. Suspect ranges require record-level comparison. Final source-to-target reconciliation occurs at a stable write barrier, with the journal applied through its recorded boundary.

The migration controller progresses through explicit states:

```text
PREPARED -> COPYING -> RECONCILING -> SHADOWING
                                      |
                                  WRITE_BARRIER
                                      |
                             TARGET_AUTHORITATIVE
                                      |
                               SOURCE_RETIRED
```

Each transition records its evidence and abort conditions. Quarantining an invalid record preserves the evidence needed to repair it; it does not count that record as successfully migrated. An unresolved correctness-critical record blocks ownership transfer.

## 4. Transfer ownership and handle ambiguous mutations

### 4.1 Define the write barrier operationally

During copying and shadowing, MongoDB remains authoritative. Shadow reads do not alter client responses. Target reads become eligible for ordinary traffic only when their declared semantics can be met; a request for the latest committed revision still follows the current authority.

The final transfer uses a bounded write barrier across the migration scope. Reads remain available, while new mutations enter a bounded durable admission queue or receive a retryable response. An asynchronous receipt means accepted for processing, not committed. Synchronous success is returned only after authoritative commit.

The barrier stops new source mutations, drains in-flight writes, and fences every old writer. That includes revoking its mutation access and terminating or draining existing write-capable sessions as required by the source deployment. The operator verifies that no old writer can commit before recording the final capture boundary. A configuration flag alone is insufficient fencing.

The target then applies through that boundary and completes reconciliation. Only afterward does the controller enable the new writer under a higher ownership epoch, disable old migration-applier writes, and switch authoritative reads. Cache and projection namespaces include the ownership epoch so that an old response cannot become a new-owner cache entry.

If the required conditions cannot be established within the admission and deadline budget, the controller aborts before granting target authority. Service availability does not imply an unconditional promise that every mutation completes without additional latency during transfer.

### 4.2 Give retries a durable result

After transfer, application mutations use optimistic concurrency against the expected record revision. A retry identifier is scoped to the tenant and operation; its stored record also binds a canonical request digest and result reference. Successful writes return the resulting `(owner_epoch, revision)` token.

For a mutation that needs durable deduplication, one DynamoDB transaction conditionally updates the object and creates its operation record. A repeated request with the same identifier and digest returns the stored result. The same identifier with different arguments is rejected. Revision conflicts and failed business preconditions remain distinct errors.

A lost response does not mean the write failed. The caller first resolves the operation record or retries through the same idempotency contract. It must not generate a new identifier merely because a timeout occurred. DynamoDB's transaction client token helps with short retries, but its documented validity window is ten minutes; a longer application contract needs its own retained record. [Transaction idempotency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)

The deduplication retention interval covers the documented client retry horizon. Once that horizon closes, an expired identifier is not silently treated as a new operation. This bounds storage without placing an indefinitely growing set of processed requests inside one frequently updated object.

### 4.3 Make rollback another controlled transfer

Before target authority, rollback can return read routing to the source. After target writes begin, the source is no longer current. These are different operations.

If post-transfer rollback is required, a restricted reverse projector keeps a losslessly representable source copy up to date while ordinary source writers remain fenced. Returning authority requires another write barrier, reverse-journal catch-up, reconciliation and a new ownership epoch. Fields that cannot be represented in the old schema make that path invalid until a compatible recovery plan exists.

A valid alternative is forward recovery, backed by tested restore procedures and an explicitly accepted recovery objective. Keeping an obsolete database online does not automatically provide a recovery capability.

## 5. Bound read work and prove behavior under load

### 5.1 Cache computation with explicit freshness

Redis holds derived responses whose assembly is expensive and whose consumers permit bounded staleness. Entries carry a projection version, authoritative object revision, ownership epoch and expiration metadata. A cache hit avoids repeated composition; it does not become evidence that a write committed.

A caller presenting a minimum `(owner_epoch, revision)` token may use an entry only if the stored version pair is sufficient and its namespace matches the active owner. Comparing revision numbers alone would fail if a new ownership generation used a different revision range. Otherwise, the service reads the authoritative record with the required consistency, recomputes if necessary, and returns a result or a bounded failure. DynamoDB strongly consistent reads are available on tables and local secondary indexes, while GSI reads are eventually consistent. [Read-consistency boundaries](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)

An aggregate involving independently updated records has an additional contract. It can expose component versions and tolerate mixed revisions, or it must use a mechanism that provides the required coherent read. Fetching several individually current values does not establish one atomic snapshot.

DAX is evaluated as an optional optimization for suitable access paths. It is not the fallback correctness mechanism for a missed Redis entry: strongly consistent reads pass through it, and its query cache has different refresh behavior from its item cache. [DAX consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.consistency.html)

The default design keeps one derived-response cache. A second layer is justified only by measured latency, capacity or cost benefits that exceed its operational and consistency burden.

### 5.2 Prevent cache failure from becoming database failure

Singleflight coalesces concurrent rebuilds for the same cache identity within one process. The identity includes tenant, object, projection version and ownership epoch. It cannot be the unique request identifier, which would prevent requests from sharing work.

Each coalesced operation has a deadline and a bounded number of waiters. Process-local coalescing does not collapse requests across all application instances, so database concurrency is also limited across the service. Tenant budgets prevent one busy tenant from occupying every rebuild slot.

When Redis fails, the service permits only the measured amount of direct database work. Remaining requests receive an allowed stale response, an asynchronous result, or an explicit overload response according to their contract. Unlimited fail-open traffic would turn a cache outage into a storage outage.

### 5.3 Attribute performance changes to specific work

The performance plan varies payload size, tenant skew, hot-object concentration, read/write mix and cache state. It includes warm operation, cold start, cache loss and backlog recovery. Offered load and achieved throughput are recorded separately.

A latency decomposition guides experiments:

```text
request latency = admission wait
                + connection and transport time
                + storage work
                + object assembly
                + serialization
```

The design changes one major source of work at a time: remove repeated joins through projections, reduce redundant reads through coalescing, bound concurrent storage calls, and batch background processing within a measured latency budget. Profiles and allocation measurements determine whether serialization or object construction deserves further work.

A single successful request proves little about capacity. Sustainable throughput requires an agreed error rate and tail-latency objective over a stated duration, including the offered workload distribution. The test harness must account for scheduled requests that were delayed or rejected, rather than measuring only the requests a saturated server happened to accept.

For background work, let `lambda` be the incoming event rate and `mu` the sustainable application rate. Recovery requires `mu > lambda`; an existing backlog of `B` events needs approximately `B / (mu - lambda)` time to drain under stable conditions. The model is simple, but it reveals why a worker fleet sized only for steady state cannot recover while new traffic continues.

Cost measurements accompany throughput: storage operations per request, index write amplification, duplicate delivery, retained journal volume, cache capacity and operator time. Faster completion purchased by unbounded fan-out is a different result from doing less work.

## 6. Deliver a service the team can recover

### 6.1 Make failure experiments release evidence

The event pipeline uses at-least-once delivery and bounded retries. With an SQS-triggered Lambda consumer, partial batch responses prevent a single failed record from unnecessarily retrying an otherwise successful batch. Visibility, redrive policy and handler behavior are configured together. [SQS consumer error handling](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)

Transient storage throttling, malformed data and version conflicts have different handling paths. Retry budgets and jitter prevent recovery workers from amplifying a capacity problem. A poison record is quarantined with its transformation version and error evidence; replay requires the repair to be identified and the result verified. [Timeouts, retry budgets and jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)

The release gate includes concrete counterexamples:

| Experiment | Required observation |
| --- | --- |
| Deliver a newer full state before an older one | The record remains at the newer version |
| Deliver the same version with a different payload | The discrepancy is surfaced as a conflict |
| Replay a live state after a newer tombstone | The object remains deleted |
| Crash after database commit and before message acknowledgement | Redelivery produces no additional logical mutation |
| Restart capture after journal persistence but before checkpoint persistence | Duplicate events remain recognizable and no captured change is omitted |
| Pause the consumer beyond supported source retention | Recovery halts or rebuilds through a defined procedure; it does not silently skip history |
| Remove Redis during peak offered load | Database concurrency stays bounded and degradation follows the interface contract |
| Attempt a source write after ownership transfer | The old writer cannot commit |

These are validation requirements for the design, not reported experiment results. A complete implementation records the observed state, request outcomes and watermarks for each test, rather than reducing the result to a screenshot of a healthy dashboard.

### 6.2 Divide ownership along interfaces

The work is divided into the writer and schema contract, capture and replay, query projections and caching, and migration validation and operations. These are ownership boundaries, not isolated implementation silos. A change to revision semantics must be reviewed by both the writer and applier owners; a projection schema change must include its rebuild and compatibility plan.

The rollout proceeds through independently useful increments: instrument the existing path, route and inventory writers, introduce versions, validate capture, backfill the target, compare query behavior, exercise the barrier, and finally transfer authority. Storage migration is never the prerequisite for fixing an obvious query or admission defect.

Release evidence is reviewable: a decision record, schema and event contracts, repeatable workload definitions, reconciliation output, failure-experiment results and a recovery runbook. Another engineer must be able to pause, resume, diagnose and restore the service without relying on the original designer's memory.

### 6.3 Keep the trade-offs visible after launch

The design accepts several costs deliberately. Full-state events consume more bandwidth to simplify replay. Sharded query paths add merge and pagination work to distribute writes. A bounded write barrier simplifies exclusive ownership at the expense of temporary mutation delay. Derived views reduce read computation while requiring explicit freshness and rebuild behavior.

Each choice has a pressure point that would justify revisiting it. Very large aggregates may require a different replication unit. Persistent single-object contention may require a business-level sequencing model. An unacceptable transfer pause may justify a more sophisticated ownership protocol. None of those extensions should be introduced merely to make the diagram more elaborate.

The project is complete when its ownership, consistency, performance and recovery claims can be checked by someone other than its designer. A replacement database is one component of that result. The durable engineering outcome is a set of explicit contracts and operating procedures that remain understandable under load, interruption and change.
