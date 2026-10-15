---
title: "Local Hybrid Retrieval for Private Knowledge Bases"
date: 2026-06-20T10:00:00+08:00
lastmod: 2026-10-15T10:00:00+08:00
draft: false
categories: ["AI Engineering"]
description: "Structured queries, inverted indexes, BM25, vector embeddings, reciprocal-rank fusion, SQLite architecture, and local privacy boundaries."
---

Different questions require different retrieval semantics. Exact quantities depend on structured queries, lexical relevance depends on inverted indexes and BM25, and conceptual similarity depends on vector embeddings. Hybrid retrieval combines these mechanisms in one auditable data path.

## Retrieval begins with query semantics

Retrieval is not a single operation. A system may need exact filtering, keyword matching, phrase location, conceptual similarity, cross-document aggregation, and time-series calculation. One vector interface cannot preserve precision, interpretability, and stability across all of these goals.

| Question type | Primary mechanism | Result meaning |
| --- | --- | --- |
| Records inside a date range | Typed filters and SQL | The complete set satisfying the predicate |
| Locations of a name or term | Full-text index and phrase query | Documents containing indexed terms |
| Passages close to a concept | Embeddings and similarity | Neighbors in a learned vector space |
| Period totals or category shares | Structured data and deterministic aggregation | Reproducible quantities |
| Lexical and conceptual recall together | Multiple retrievers and rank fusion | A merged evidence candidate set |

Retrieval output usually represents candidate evidence rather than a complete fact set. A top-k list means “the first k items under a scoring function,” not “no other relevant material exists.”

## Structured queries preserve exact facts

Relational tables, CSV data, and typed records support equality predicates, ranges, grouping, ordering, and aggregation. Dates, currencies, amounts, states, and categories carry explicit data types and calculation rules. Correctness for these questions comes from schemas, constraints, and deterministic computation rather than textual similarity.

Vector retrieval returns records that appear close in semantic space. It does not guarantee complete row coverage for a period, decimal precision, accounting identities, or deduplication rules. Quantitative reports therefore originate from structured data, while text retrieval discovers explanatory material and context.

A hybrid system does not convert every datum into a vector. Each data type retains its original computational semantics, and the query layer combines outputs where appropriate.

## Inverted indexes and full-text search

A normal document representation maps each document to its terms. An inverted index reverses that relation. Each term points to a list of documents or positions where the term occurs; these lists are commonly called postings. A term query reads the corresponding postings instead of scanning every document.

**Source documents**: Full text and metadata remain available

**Tokenization**: Text becomes normalized terms

**Postings**: Terms point to documents and positions

**Query**: Posting lists are intersected or merged

**Ranking**: Relevant candidates are ordered

Position data enables phrase and proximity queries. Field indexes allow different weights for titles, bodies, or labels. English tokenization commonly follows word boundaries. Chinese text has no explicit whitespace boundaries, so implementations often use dictionaries, statistical segmenters, or character n-grams. Character bigrams and trigrams increase index size but remain stable for new terms, abbreviations, and mixed-language text.

Full-text search describes the overall capability to query complete text. An inverted index describes the central data structure behind that capability. The concepts frequently appear together but occupy different abstraction levels.

## How BM25 ranks lexical matches

BM25 is a term-statistics relevance function. Its main factors are term frequency inside a document, term rarity across the corpus, and document length relative to the corpus average. A common expression is:

score(D,Q) = Σ IDF(t) × f(t,D)(k₁+1) / [f(t,D) + k₁(1-b+b|D|/avgdl)]

- `IDF(t)` increases the weight of rare terms and reduces the influence of ubiquitous terms.
- `f(t,D)` represents term frequency; saturation prevents ten repetitions from carrying ten times the effect of one repetition.
- `|D|/avgdl` normalizes length and reduces the advantage of long documents that contain more words by default.
- `k₁` and `b` control term-frequency saturation and length normalization.

BM25 remains directly interpretable for proper nouns, error codes, product names, and source wording. Its main limitation is vocabulary mismatch. Two passages can express the same concept without sharing indexed terms.

## Vector embeddings and semantic similarity

An embedding model maps text to a fixed-dimensional numeric vector. Training objectives place semantically related text closer in the learned vector space. Retrieval commonly uses cosine similarity, dot product, or Euclidean distance.

cosine(a,b) = (a · b) / (||a|| × ||b||)

An exact vector scan compares a query vector with every candidate. Results are deterministic, but computation grows linearly with vector count. Approximate-nearest-neighbor structures such as HNSW and IVF exchange additional memory, build cost, and some recall for lower latency. Exact scans have lower operational complexity at smaller scales; approximate indexes become relevant as scale and latency constraints increase.

Vector similarity does not establish factual identity. Models can confuse negation, numbers, time, entities, or causal direction. Long-text truncation and chunk boundaries also change representations. Vector retrieval therefore expands recall, while complete source text remains the basis for evidence confirmation.

## Hybrid retrieval and RRF

Lexical scores and vector similarities use different scales. Direct addition introduces calibration and scaling instability. Reciprocal Rank Fusion (RRF) uses rank positions instead:

RRF(d) = Σ 1 / (k + rank(d))

Each retriever produces an independent ranked list. A document receives a larger fused score when it appears near the top of multiple lists. The constant `k` reduces sensitivity to small rank differences at the head. RRF combines heterogeneous retrievers without calibrating BM25 and cosine values to a shared numeric interval.

Fusion remains bounded by each retriever's candidate set. Required structured evidence absent from every candidate set cannot reappear through fusion. Mandatory evidence sets, metadata filters, and multi-channel retrieval operate at separate control layers.

## SQLite as a local retrieval layer

SQLite is an in-process database with no separate database server. FTS5 provides full-text indexes, phrase queries, highlighting, snippets, and `bm25()`. Document metadata, chunks, build versions, and vector BLOBs can reside in one database file. Transactions and atomic file replacement provide a clear publication boundary.

| Table or index | Responsibility |
| --- | --- |
| `documents` | Source identifier, title, date, domain, and content hash |
| `chunks` | Stable chunks, headings, order, and source fragments |
| `FTS5` | Term postings and BM25 ranking |
| `embeddings` | Model identity, dimensions, and float32 vectors |
| `metadata` | Schema version, source fingerprint, and build counts |

SQLite provides simple deployment, direct backups, SQL filtering, and local transactions. Constraints include coordination around a single writer, linear cost for large exact vector scans, and the absence of built-in database-file encryption in the standard distribution. File permissions, full-disk encryption, backup placement, and process privileges remain part of the system security model.

## Source authority and privacy boundaries

Separating derived indexes from authoritative data reduces the mutation surface. An indexer reads source material, while databases, models, temporary files, locks, and logs occupy a separate cache layer. Deleting the index leaves source material intact, and retrieval output never writes back into source records.

**Authoritative sources**Documents and structured records

**Read-only extraction**Allowlist, parsing, and hashes

**Derived index**FTS, vectors, and metadata

**Candidate retrieval**Filters, ranking, and channel provenance

**Evidence confirmation**Complete authoritative source reopened

A local database does not create privacy by itself. Embedding software may download public model weights during initial setup; cached local inference does not require sending text to a remote model API. Temporary files, crash dumps, shell history, synchronized folders, and backups can still expand the data boundary. Security properties emerge from the complete data flow and runtime environment rather than from a file extension.

## Index build and update lifecycle

1. An allowlist configuration defines and normalizes source paths.
2. Source membership and content hashes form a pre-build fingerprint.
3. Headings, paragraphs, and length limits define chunks; source path, date, heading, and chunk hash remain attached.
4. A new temporary database receives FTS5 data and vectors. Unchanged chunk identities can reuse vectors from the previous database.
5. A second source fingerprint detects concurrent source changes. A mismatch discards the temporary result.
6. Database integrity checks, file synchronization, and atomic replacement publish the new version while the old version remains queryable until replacement.

Stable chunk identities enable incremental reuse. Changes to model name, dimensions, chunking rules, or tokenization alter index semantics. Build metadata identifies these incompatibilities. A single build lock prevents concurrent writers from publishing conflicting versions.

## Evaluation and failure modes

Retrieval evaluation uses a query set with relevance judgments. Recall@k measures whether relevant items enter the first k results. MRR emphasizes the rank of the first relevant item. nDCG handles graded relevance. Answer accuracy, citation completeness, and reasoning reliability form a separate post-retrieval evaluation layer.

| Failure mode | Observable effect | Control |
| --- | --- | --- |
| Chinese token mismatch | Short terms, new terms, or mixed-language terms disappear | n-grams, domain lexicons, query analysis |
| Chunks too large | Mixed topics and imprecise source locations | Heading-aware paragraph chunks |
| Chunks too small | Lost context, negation, or entities | Overlap windows and parent-document reads |
| False vector similarity | Similar wording but different facts | Lexical cross-checks, metadata filters, source confirmation |
| Stale index | Updated sources still return old fragments | Source fingerprints, build versions, freshness checks |
| Top-k treated as a complete set | Missing aggregates and false absence claims | Structured queries, full enumeration, required evidence sets |

## Alternative storage and retrieval systems

| System | Main characteristics | Typical conditions |
| --- | --- | --- |
| SQLite + FTS5 + vector BLOBs | Single file, in-process, exact scans | Local workloads, low operations, limited scale and concurrency |
| sqlite-vec | SQLite vector extension with vector queries inside SQL | Single-file architecture with additional vector SQL operations |
| PostgreSQL + pgvector | Transactional database, rich filters, HNSW and IVFFlat | Existing PostgreSQL deployment, concurrent writes, service queries |
| Qdrant | Dedicated vector service, filtering, distributed API | High-concurrency APIs and an independent service lifecycle |
| LanceDB | Embedded columnar and multimodal retrieval | Larger analytical datasets, object storage, or multimodal workloads |

Selection criteria include data scale, write concurrency, latency targets, filter complexity, backup design, encryption requirements, and operational capacity. A vector database handles vector storage and neighbor search. It does not replace source governance, deterministic calculations, evidence citations, or privacy controls.

## Sources and notes

- [SQLite FTS5 Extension](https://www.sqlite.org/fts5.html): FTS5 queries, tokenizers, ranking, and `bm25()`.
- [SQLite Is Serverless](https://www.sqlite.org/serverless.html): in-process operation without a separate database server.
- [The Probabilistic Relevance Framework: BM25 and Beyond](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf): the probabilistic relevance framework behind BM25.
- [Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods](https://doi.org/10.1145/1571941.1572114): reciprocal-rank fusion.
- [Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs](https://arxiv.org/abs/1603.09320): HNSW.
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401): the retrieval-augmented generation architecture.
- [FastEmbed](https://github.com/qdrant/fastembed), [sqlite-vec](https://github.com/asg017/sqlite-vec), [pgvector](https://github.com/pgvector/pgvector), [Qdrant](https://github.com/qdrant/qdrant), and [LanceDB](https://github.com/lancedb/lancedb): implementation references for local embeddings and vector storage.

This article describes general information-retrieval architecture. It is not a security audit, compliance opinion, or capacity plan for a specific dataset. Component versions, extension maturity, and performance characteristics change over time.
