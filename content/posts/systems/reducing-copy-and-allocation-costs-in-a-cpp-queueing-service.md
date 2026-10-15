---
title: "Reducing Copy and Allocation Costs in a C++ Queueing Service"
date: 2022-08-20T10:00:00+08:00
draft: false
categories: ["Software Engineering"]
description: "A focused C++ performance case study: profile expensive queue transitions, establish single ownership, reject stale handles, and validate lifecycle behavior before rollout."
---

A queue that stores objects by value is easy to understand while the objects are small. As each record accumulates strings, configuration, and nested collections, moving it through several eligibility queues can become more expensive than deciding where it belongs. The scheduling algorithm remains unchanged, but the cost of expressing its decisions grows with the payload.

Consider a C++ service that groups compatible work before dispatch. Requests can enter a queue, move to another eligibility band, time out, or be cancelled. This case study focuses on the in-memory scheduling module: reduce repeated object copying while preserving the existing selection rules and lifecycle behavior.

```text
Before
  admission -> queue A [full records]
                        |
                   copy / destroy
                        v
               queue B [full records] -> dispatch

After: one scheduler thread owns mutable state
  commands -> bounded mailbox -> record registry
                                  id -> unique owner
                                         ^
                                         | validated lookup
                           queues [small handles]
                                         |
                            consume / cancel / expire
```

This is a worked design case with illustrative C++17 code and a validation plan. It does not claim measured production speedups. The scope is a local module and its deployment behavior, rather than a redesign of routing, durable storage, or distributed dispatch.

## 1. Find the cost of a queue transition

### 1.1 Profile the service with representative object sizes

The initial symptom is higher processing latency as request records become larger. One hypothesis is that the selection algorithm has become expensive. Another is that the algorithm performs the same work while repeatedly constructing and destroying larger objects. Those explanations call for different changes.

A sampling CPU profile can identify time in copy constructors, string operations, allocation, and queue manipulation. Allocation profiles then help distinguish repeated short-lived buffers from long-lived retained records. The [gperftools CPU profiler documentation](https://gperftools.github.io/gperftools/cpuprofile.html) describes the sampling and call-graph workflow; the profile should come from representative traffic rather than an empty-record benchmark.

The test workload needs the distribution of payload sizes, transitions per request, cancellation frequency, and waiting time. A median-sized object moved once can conceal the cost of a large object moved repeatedly. It is also useful to instrument the record's copy constructor in a test build, because a profile alone may not reveal which container operation introduced a copy.

Memory measurements need similar care. High resident memory is not sufficient evidence of allocator fragmentation. Separate live object bytes, allocation counts, allocator-retained memory where available, and process RSS. A heap profiler run can explain allocation behavior, but its timings should not be compared directly with an uninstrumented baseline.

The desired output is a causal chain: a queue transition copies nested state, that copy creates allocations and memory traffic, and transition frequency makes the cost material. Without that chain, changing the allocator or replacing every pointer type would be guesswork.

### 1.2 Define the behavior that must stay the same

The optimization preserves eligibility rules, timeout deadlines, and the original waiting timestamp. Moving a request between internal queues must not reset its age and quietly change fairness. Existing instance routing remains in place so that a throughput change cannot be confused with a change in which requests meet each other.

Three invariants make the review concrete. Each accepted request has one authoritative record. A request has at most one active scheduling ticket in this simplified design. A terminal request cannot be dispatched by a late ticket or callback.

The second invariant is a deliberate scope choice. Multiple indexes may observe a request, but they do not each need to own it. If the real algorithm requires simultaneous active membership in several queues, the ticket model must become per-membership and the tests must cover invalidating all memberships. That extension should not be smuggled into the implementation as an assumption.

## 2. Give the record one owner and the queues small handles

### 2.1 Separate ownership from membership

A straightforward registry is `unordered_map<Id, unique_ptr<Entry>>`. The scheduler creates an entry on admission and erases it after a terminal transition. Queues store small values identifying the entry rather than copies of its payload. The registry is per scheduler instance, not a process-wide mutable singleton.

This is an ownership registry, not automatically an object pool. Each admission can still allocate the record and its nested fields. Queue storage can also allocate. The intended improvement is narrower: an eligibility transition no longer copies the large record. Further pooling should be justified by a new allocation profile after this change.

`unique_ptr` makes the owning relationship explicit and destroys the owned object when that ownership ends, as specified by the [C++20 working draft's unique pointer clauses](https://timsong-cpp.github.io/cppwp/n4861/unique.ptr). The example itself uses C++17 features. Queue membership does not require shared ownership simply because several parts of an algorithm refer to one record.

An alternative is `shared_ptr<Entry>` in every queue. That can be reasonable when independent asynchronous operations genuinely own a record's lifetime. Here it would make deletion less predictable: removing the registry entry would not free a record still retained by a queue. It would also leave mutable access synchronization as a separate problem. Reference counting alone does not establish the scheduler's concurrency rules.

### 2.2 Use identity that survives stale queue entries

A bare identifier is insufficient when identifiers can be reused. Suppose request 42 is cancelled, another request with identifier 42 is admitted, and an old queue entry is processed afterward. A lookup by identifier alone finds a valid record of the wrong lifetime.

A handle therefore includes an admission generation and a ticket number:

```cpp
using Id = std::uint64_t;

struct Handle {
    Id id;
    std::uint64_t generation;
    std::uint64_t ticket;
};

struct Entry {
    std::uint64_t generation;
    std::uint64_t ticket = 0;
    bool queued = false;
    Payload payload;
    TimePoint admitted_at;
    TimePoint expires_at;
};

std::unordered_map<Id, std::unique_ptr<Entry>> records;
```

The scheduler assigns a process-wide monotonically increasing generation at admission. Before wraparound, it refuses further admissions and drains for restart; correctness does not depend on overflow being unlikely. Internal handles never survive a process restart. Any externally delivered callback also needs an instance-incarnation identifier so a callback from an old process cannot accidentally match a new generation.

A queue transition increments the ticket and invalidates older membership. A cancellation removes the record. A later admission with the same identifier receives a different generation. These values solve different problems: generation distinguishes record lifetimes, while ticket distinguishes queue memberships within one lifetime.

This adds lookup and validation work to a dequeue. That cost is the explicit tradeoff for avoiding payload copies and unsafe borrowed references. Whether it wins depends on the actual payload and transition distribution, which the benchmark must preserve.

## 3. Make cancellation and expiry ordinary state transitions

### 3.1 Confine mutation to the scheduler thread

The simplest concurrency model for this change is one owner thread per existing scheduler instance. Network handlers and timers submit commands through a bounded mailbox; they do not read or mutate registry entries directly. The owner processes queue transitions, cancellations, and dispatch decisions sequentially.

A single owner does not make mailbox access automatically safe. The mailbox needs an appropriate synchronized implementation, and its memory bound must include payload bytes as well as command count. Admission can be rejected under pressure. Cancellation commands use reserved bounded capacity and an explicit retry or expiry policy so that ordinary admissions cannot indefinitely crowd out cleanup.

This model also bounds what a pointer can mean. A temporary `Entry*` may be used within one owner-thread operation after validation. It is not stored in callbacks or retained across an asynchronous wait. A worker receives the immutable input it needs and returns a handle with its result. The owner validates the handle again before applying that result.

The design may still copy selected data when crossing a thread or network boundary. That is preferable to describing the entire system as zero-copy when the actual optimization concerns internal queue transitions.

### 3.2 Consume a ticket before doing terminal work

The central lookup rejects absent records, old admissions, and old memberships. Consuming a valid ticket invalidates duplicate copies of that same ticket before dispatch proceeds.

```cpp
Entry* consume(const Handle& h) {
    auto it = records.find(h.id);
    if (it == records.end()) return nullptr;

    Entry& e = *it->second;
    if (e.generation != h.generation ||
        e.ticket != h.ticket || !e.queued) {
        return nullptr;
    }

    e.queued = false;
    return &e; // Owner thread only; never retain across a callback.
}
```

A dequeue first validates and consumes the ticket, then checks the authoritative expiry time before selecting or dispatching. If it cannot proceed, it creates a new ticket and returns the request to the appropriate queue without changing `admitted_at`. Ticket overflow is handled as an error before incrementing, not allowed to wrap into an old valid value.

Cancellation validates the admission generation and erases the record. It does not need to find every stale queue handle immediately. A timer carries the same admission identity, and an old timer cannot erase a replacement request. A late asynchronous result must additionally match the expected operation ticket and phase; checking only the identifier and generation would accept work from an earlier attempt within the same admission.

When dispatch and cancellation race, owner-thread processing order determines the local outcome. If dispatch has already committed the local terminal transition, a later cancellation reports that the request is no longer queued. It cannot promise to undo an external operation. Exactly-once execution across process crashes would require a separate durable protocol, outside this module's contract.

### 3.3 Do not let lazy invalidation become a memory leak

Stale handles are small, but their number can still grow without bound if requests move repeatedly and old queues are rarely inspected. Lazy invalidation therefore needs maintenance work and an admission bound.

The scheduler tracks live records, total queued handles, stale handles discarded, and oldest queue age. A maintenance pass removes invalid handles incrementally. When the handle budget is approached, the scheduler compacts affected queues before allowing more transitions; if it cannot make progress within its work budget, it applies backpressure rather than allocating indefinitely.

Compaction preserves the ordering keys used by the existing selection policy. Rebuilding a queue from arbitrary hash-map iteration order would silently alter fairness. Maintenance also has a per-tick work cap so that reclaiming memory cannot monopolize the owner thread and cause the latency spike the optimization was meant to remove.

An intrusive queue with explicit membership nodes could provide direct removal instead. It avoids stale-handle accumulation but requires more complex node lifetime management, especially with multiple indexes. For this bounded module, validated handles are a reasonable first implementation if churn tests demonstrate that cleanup stays within its budget.

## 4. Verify lifetime behavior before comparing throughput

### 4.1 Test the event orderings that create stale references

A deterministic test harness can drive the owner without real sockets or wall-clock sleeps. It advances a fake clock and submits admission, movement, cancellation, expiry, and completion commands. The reference implementation retains the original selection rules and provides a behavior oracle for valid dispatches.

The important cases are sequences, not isolated methods:

| Sequence | Required observation |
| --- | --- |
| Admit, move, process old ticket | Old ticket is rejected; current membership remains valid |
| Admit, cancel, reuse identifier | Old handle cannot access the replacement record |
| Admit, enqueue same handle twice, consume twice | At most one consume succeeds |
| Admit, timeout, receive delayed completion | Completion does not recreate or dispatch the record |
| Move several times, compact queues | Selection order and original waiting age are preserved |
| Repeated cancellation under load | Registry and handle storage return to a bounded baseline |

Randomized command sequences can check that dispatched identifiers are never also live and queued in the same admission, that expired requests are not selected, and that registry size matches the reference model. A reproducible seed makes a failure debuggable. A long soak test supplements these checks but cannot prove that a dangling reference is impossible.

Memory-safety runs should use AddressSanitizer to catch errors such as use-after-free and out-of-bounds access. Its documented capabilities and limitations are described in the [Clang AddressSanitizer guide](https://clang.llvm.org/docs/AddressSanitizer.html). Race detection is a separate run for mailbox and callback integration. Neither replaces the generation and ticket invariants: a logically stale handle can be memory-safe while still selecting the wrong request.

### 4.2 Measure the complete scheduling cost

The performance comparison has two levels. A transition benchmark isolates the copy cost using realistic record sizes. An integrated instance benchmark includes admissions, queue churn, timeouts, cancellations, and dispatch work. Both versions use the same compiler options, allocator, selection rules, and input sequence.

Track CPU time per completed request, allocation count and bytes, live record memory, RSS after a drain, and scheduling latency. Tail latency should include requests waiting for the scheduler, not just time spent inside its selection function. An offered-load test also records rejected and timed-out requests so that overload cannot masquerade as better latency.

The interpretation needs to remain precise. Fewer allocations during transitions would support the ownership change. Lower RSS after a drain might indicate reduced retention, but would not by itself prove fragmentation was eliminated. If lookup or stale-handle scanning dominates after the change, the profile has identified a new tradeoff to address.

Allocator replacement is a separate experiment. Combining a new allocator, different queue representation, and new routing in one release would make both performance attribution and rollback harder. The first release should establish whether removing payload copies is enough.

## 5. Roll out the module without losing in-flight requests

The implementation can be reviewed in small parts: profiling instrumentation, explicit registry ownership, validated handles, lifecycle tests, and finally routing a small cohort to the new module. The work includes code review of every place that previously copied a record or retained a pointer. A conversion that misses one callback can pass throughput tests and still fail under cancellation.

Existing requests stay with the implementation that admitted them. New admissions are assigned to one implementation; their cancellations and timeouts follow that ownership. The system does not insert one request into both live schedulers to compare results, since both could dispatch it. A shadow scheduler may compare decisions only when all external side effects are disabled and the additional CPU budget is acceptable.

Rollback stops new admissions to the new module and drains its existing records. After a bounded drain interval, remaining requests receive the service's established timeout or retry response. Requests are not silently copied into the old module without an explicit transfer protocol. This keeps the release mechanism consistent with the single-owner model.

The module is stateful even if its queued intent is temporary. A process crash loses in-memory scheduling state. Recovery relies on the surrounding caller contract to reconcile or retry intent; this refactor does not add durable recovery or claim that arbitrary retries are harmless. That boundary belongs in the runbook alongside the drain procedure.

The handoff should include the lifecycle diagram, handle-validation rules, benchmark inputs, and counters for live records, stale handles, owner-thread lag, and rejected commands. Reviewers can then assess both the performance hypothesis and the failure behavior without reconstructing ownership from scattered callbacks.

There are several reasons to keep the change this narrow. Single-thread ownership simplifies lifetimes but leaves a per-instance CPU ceiling. Registry lookups cost more than direct access to a compact value queue when records are tiny. Lazy cleanup saves removal complexity but requires a hard storage bound. A dedicated allocator or a different routing strategy may eventually be useful, yet neither is necessary to establish a correct ownership model.

The next decision follows from the post-change profile. If payload copying disappears and selection itself becomes dominant, algorithm work has a clearer justification. If owner-thread lag comes from cleanup, queue representation deserves another pass. The useful result is a module whose per-request costs and lifecycle can be explained together, rather than a throughput number detached from how requests actually leave the system.
