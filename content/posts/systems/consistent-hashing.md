---
title: "Consistent Hashing for Distributed Systems"
date: 2021-09-09T00:00:00+08:00
draft: false
categories: ["Distributed Systems"]
description: "How hash rings, Jump Consistent Hash, and Maglev balance distribution, remapping, lookup cost, and operational constraints."
aliases:
  - /posts/service-governance/load-balancing/
---

# Consistent Hashing for Distributed Systems

Many distributed systems need a deterministic answer to a simple question: given a key, which node should own it?

A stateless load balancer can choose the least-loaded backend for every request. A cache, shard map, or stateful service cannot. Sending the same key to different nodes destroys locality, while moving too many keys during a resize can overload storage and downstream databases.

Consistent hashing is a family of techniques for balancing four goals:

1. distribute keys evenly;
2. map the same key to the same node;
3. move as few keys as possible when membership changes;
4. keep lookup and memory costs practical.

There is no universally best algorithm. Hash rings, Jump Consistent Hash, and Maglev make different trade-offs because they solve different operational problems.

## Why modulo hashing fails during a resize

The most direct mapping is:

```text
owner(key) = hash(key) mod N
```

For a stable set of `N` nodes, this is fast, deterministic, and usually balanced when the hash function is well distributed. The problem appears when `N` changes.

Suppose a cache grows from four nodes to five. A key remains on the same numbered node only when both of these expressions agree:

```text
hash(key) mod 4
hash(key) mod 5
```

Most keys change owners. In a cache, that produces a sudden miss storm. In a storage system, it requires a near-global redistribution. The membership change is small, but its consequence is not.

Consistent hashing replaces the dependency on the total node count with a mapping whose local structure changes incrementally.

## The properties that matter

The original consistent-hashing work described a broader set of properties, but four are especially useful in practice:

- **Balance:** ownership should be distributed evenly across nodes.
- **Monotonicity:** adding a node should move keys to the new node, not arbitrarily between existing nodes.
- **Minimal disruption:** membership changes should remap only a bounded fraction of keys.
- **Load control:** no node should receive far more keys than its capacity permits.

These properties can conflict. A method with extremely cheap lookups may require a large precomputed table. A method with constant memory may support only a restricted form of membership change. Operational constraints—not elegance alone—should decide the algorithm.

## Hash rings and virtual nodes

The best-known construction maps both nodes and keys into the same circular hash space.

```text
                 node B
                   ●
             .-----------.
          .-'             '-.
 node A ●                     ● node C
          '-.             .-'
             '-----------'

key -> first node encountered clockwise
```

To place a key:

1. hash the key onto the ring;
2. move clockwise;
3. select the first node position encountered.

When a new node is inserted, it takes responsibility for the interval between its predecessor and itself. Other intervals do not change. Removing a node transfers its interval to its successor.

With node positions stored in a sorted array or tree, a lookup is a successor search and costs `O(log V)`, where `V` is the number of positions on the ring.

### Why one position per node is not enough

Randomly placing a small number of physical nodes creates uneven intervals. One node may own a large arc while another owns very little.

Virtual nodes reduce this variance. Each physical node is represented by many independently hashed positions. Ownership from those positions is aggregated back to the physical node.

Virtual nodes also support capacity weighting. A node with twice the intended capacity can receive approximately twice as many virtual positions. This is convenient, but it introduces configuration and memory costs. The number of virtual nodes must be large enough for smooth distribution without making membership tables unnecessarily heavy.

Hash rings are a strong general-purpose choice when nodes have stable identities, membership can change at arbitrary positions, and weighted capacity matters.

## Jump Consistent Hash

Jump Consistent Hash takes a different approach. Given a key and an integer number of buckets, it calculates the bucket directly without storing a ring or lookup table.

Its design follows one observation: when the system grows from `n` buckets to `n + 1`, exactly `1 / (n + 1)` of the keys should move to the new bucket to preserve balance. A deterministic pseudorandom sequence derived from each key decides whether and when that key "jumps" to a newly added bucket.

The published algorithm uses constant memory and expected `O(log N)` time. It provides excellent balance and minimal movement when buckets are appended or removed from the end of a dense numeric range.

That last condition is the important limitation. Bucket identity is its index. Removing an arbitrary bucket from the middle renumbers later buckets and breaks the intended movement property. Production systems commonly add an indirection layer from stable node identities to dense bucket numbers, but managing that layer becomes part of the system design.

Jump Consistent Hash is attractive when:

- membership changes can be represented as changes to a dense bucket count;
- all buckets have equal weight, or weighting is handled elsewhere;
- constant memory and small client state are important.

## Maglev hashing

Maglev was designed for Google's network load balancers. Its goal is fast packet-level backend selection with consistent behavior across many load-balancer instances.

For each backend, Maglev derives a deterministic permutation of a fixed-size lookup table. It then interleaves those permutations to populate the table as evenly as possible. Request lookup is simple:

```text
backend = table[hash(key) mod M]
```

The hot-path lookup is therefore `O(1)`. Every load balancer that sees the same ordered backend set can construct the same table, so flows are mapped consistently without sharing per-flow state.

Membership changes require rebuilding the table. Maglev aims for small disruption, but unlike the idealized ring model it does not guarantee that only keys owned by the changed backend move. Its strength is the combination of high lookup speed, good balance, compact runtime behavior, and deterministic table construction.

Table size `M` is a design parameter. The original paper recommends a prime number larger than the product of the expected maximum backend count and a factor chosen for balance. Larger tables improve distribution and reduce collision effects but cost more memory and rebuild time.

Maglev is a strong fit for high-throughput load balancing where the backend set is replicated to many data-plane processes and lookup latency dominates.

## Comparing the algorithms

| Algorithm | Lookup | Runtime state | Membership model | Weighting | Best fit |
| --- | --- | --- | --- | --- | --- |
| Modulo hash | `O(1)` | None | Fixed membership | External | Static sharding |
| Hash ring | `O(log V)` | Ring positions | Arbitrary node add/remove | Virtual nodes | Caches and general sharding |
| Jump hash | Expected `O(log N)` | Constant | Dense bucket count | External | Compact client-side placement |
| Maglev | `O(1)` | Lookup table | Rebuilt backend set | Extensions required | High-throughput load balancing |

The asymptotic lookup cost is only part of the decision. A production review should also ask:

- How is membership distributed and versioned?
- Can two clients temporarily disagree about the node set?
- Are nodes equal, or do they need capacity weights?
- What happens to in-flight requests during a remap?
- Is the key distribution adversarial or naturally uniform?
- Is a moved key a cache miss, a network retry, or a data migration?

## Operational caveats

### Membership consistency

Deterministic hashing produces consistent results only when participants use the same inputs. Two clients with different membership snapshots can route the same key to different owners. Configuration distribution, epochs, and rollout sequencing are therefore part of the hashing system.

### Hot keys

Even a perfectly balanced key count does not imply balanced traffic. One key may be responsible for a large share of requests. Replication, request coalescing, hot-key splitting, or a secondary load-aware policy may still be necessary.

### Bounded loads

Classic consistent hashing controls movement, but it does not strictly cap the largest node load. Systems with tight capacity requirements may use bounded-load variants or choose a small set of candidate nodes and then consider observed load.

### Stateful movement

Hashing decides the new owner; it does not move data safely. Storage systems still need replication, handoff, consistency rules, and failure recovery. A mathematically minimal remap can still be operationally dangerous if all migrations begin at once.

### Hash function choice

Cryptographic strength is usually unnecessary for placement. Distribution quality, speed, stable cross-language implementation, and resistance to untrusted-key attacks matter more. The hash output and byte encoding must be specified precisely if different clients need identical results.

## Choosing deliberately

Use modulo hashing when membership is genuinely static or when a full reshuffle is acceptable. Use a hash ring when arbitrary membership, stable node identity, and capacity weighting are central. Use Jump Consistent Hash when constant memory and a dense bucket model match the system. Use Maglev when a precomputed table buys the simplest and fastest request path.

The broader lesson is that "consistent hashing" is not a single algorithm. It is a design space shaped by the cost of movement, the topology of membership changes, the location of state, and the latency budget of each lookup.

## References

- David Karger et al., [Consistent Hashing and Random Trees](https://dl.acm.org/doi/10.1145/258533.258660), STOC 1997.
- John Lamping and Eric Veach, [A Fast, Minimal Memory, Consistent Hash Algorithm](https://arxiv.org/abs/1406.2294), 2014.
- Daniel E. Eisenbud et al., [Maglev: A Fast and Reliable Software Network Load Balancer](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/), NSDI 2016.

## Original references

- [Reference 1](https://en.wikipedia.org/wiki/Hash_table#Collision_resolution)
- [Reference 2](https://alibaba-cloud.medium.com/struggling-with-poor-responsiveness-unlock-the-power-of-caching-b3186f2b3cd0)
- [Reference 3](https://www.metabrew.com/article/libketama-consistent-hashing-algo-memcached-clients#:~:text=Ketama%20is%20an%20implementation%20of,complete%20remap%20of%20all%20keys)
- [Reference 4](https://en.wikipedia.org/wiki/Pseudorandom_number_generator)
- [Reference 5](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44824.pdf)
- [Reference 6](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)

## Figures from the original edition

![Figure 1 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/reverse-proxy/reverse-proxy.png)
![Figure 2 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/separate-chaining.png)
![Figure 3 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/open-addressing.png)
![Figure 4 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/hash-ring-add-1.png)
![Figure 5 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/hash-ring-add-2.png)
![Figure 6 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/virtual-slot.png)
![Figure 7 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/jump-consistent-hashing-0.png)
![Figure 8 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/jump-consistent-hashing-1.png)
![Figure 9 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/karger.png)
![Figure 10 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/maglev-table.png)
![Figure 11 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/permutation.png)
![Figure 12 from the original edition](https://raw.githubusercontent.com/chr1sc2y/warehouse-deprecated/refs/heads/main/resources/consistent-hashing/minimal-disruption.png)
