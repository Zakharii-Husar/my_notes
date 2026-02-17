# Computer Science Cheatsheet (High-Level Coverage)

---

## 0) Mental Map

- **Theory**: math, models of computation, complexity, proofs
- **Systems**: OS, networking, distributed systems, databases, compilers
- **Software**: design, architecture, testing, security, performance
- **Data**: data structures, algorithms, ML/AI, data engineering
- **Practice**: tooling, reliability, observability, delivery

---

## 1) Discrete Math (Core Language of CS)

- **Logic**: propositions, predicates, implication, De Morgan, quantifiers
- **Proofs**: direct, contradiction, contrapositive, induction, invariants
- **Sets**: union/intersection, relations, functions, equivalence relations
- **Combinatorics**: permutations/combinations, pigeonhole principle
- **Graphs**: trees, DAGs, cycles, connectivity, bipartite, matching
- **Number theory**: modular arithmetic, gcd, primes (crypto basics)

---

## 2) Algorithms & Complexity

### Big-O + costs

- Time/space tradeoffs, amortized analysis, constant factors, cache effects
- Common classes: `O(1) O(log n) O(n) O(n log n) O(n^2) O(2^n) O(n!)`

### Design patterns

- Divide & conquer, greedy, dynamic programming, backtracking, memoization
- Two pointers, sliding window, prefix sums, hashing, heaps, union-find
- Randomized algorithms, approximation algorithms

### Core problems

- Sorting/searching, selection (kth), string matching
- Graph traversal (BFS/DFS), shortest paths, MST, topological sort
- Flows/matching, connectivity, SCC

### Complexity theory (high level)

- **P vs NP**, NP-complete, reductions
- **NP-hard** vs NP-complete
- Decidability (halting problem)

---

## 3) Data Structures (What you should “feel” in your bones)

- Arrays, dynamic arrays, linked lists
- Stacks/queues/deques
- Hash tables: collisions (chaining/open addressing), load factor, rehashing
- Trees: BST, AVL/Red-Black, B-Tree/B+Tree (databases), tries (strings)
- Heaps / priority queues
- Graph representations: adjacency list/matrix
- Bloom filters, skip lists
- LSM trees (storage engines), roaring bitmaps (analytics)
- Persistent/immutable DS (functional style)
- Concurrency-aware: lock-free queues, concurrent hash maps (conceptually)

---

## 4) Programming Language Concepts

- Compilation vs interpretation, JIT, bytecode, VMs
- Static vs dynamic typing; soundness; inference
- Memory: stack vs heap; pointers/references; aliasing
- Value vs reference semantics; copy vs move
- Closures, higher-order functions, recursion
- Polymorphism: parametric vs subtype; generics
- Error models: exceptions, result types, panic/abort
- Concurrency model: threads, async/await, actors
- Determinism vs side effects; purity; referential transparency
- TypeScript/JS-specific gotchas: event loop, microtasks vs macrotasks, prototype chain

---

## 5) Operating Systems (The Stuff Under Your App)

### Processes & threads

- Process isolation, context switch, scheduling
- Threads vs processes; user vs kernel threads
- Concurrency hazards: races, deadlocks, starvation, livelock

### Memory

- Virtual memory, paging, page faults
- MMU, address spaces, shared memory
- Caches: CPU cache lines, locality, false sharing
- Allocation: malloc/new, GC vs manual, fragmentation

### I/O

- Syscalls, file descriptors, buffering
- Blocking vs nonblocking, async I/O, epoll/kqueue
- Filesystems: journaling, inodes, permissions

### Synchronization primitives

- Mutex, RW lock, semaphore, condition variable
- Atomics, CAS, memory ordering (high-level awareness)

---

## 6) Networking (Web Dev’s “Real World”)

### Layers (roughly)

- L2: Ethernet
- L3: IP, routing, NAT
- L4: TCP/UDP, ports
- L7: HTTP, DNS, TLS, WebSocket, gRPC

### TCP essentials

- 3-way handshake, congestion control, retransmits
- HOL blocking, Nagle, slow start (performance implications)

### HTTP

- HTTP/1.1 vs HTTP/2 vs HTTP/3(QUIC)
- Caching headers: `Cache-Control`, `ETag`, `Vary`
- Content negotiation, compression, chunked encoding
- Idempotency, retries, timeouts, backoff

### DNS

- Recursive resolvers, TTL, CNAME/A/AAAA, split-horizon
- Common issues: propagation, caching, stale records

### TLS

- Certificates, chains, SNI, ALPN, forward secrecy
- mTLS for service-to-service auth

### WebSocket/SSE

- WS: full duplex
- SSE: server → client stream; simple; HTTP-friendly

---

## 7) Databases (Model + Storage + Consistency)

### Data modeling

- Entities/relations, normalization vs denormalization
- OLTP vs OLAP
- Constraints, keys, indexes, cardinality

### SQL fundamentals

- Joins, grouping, window functions, transactions
- Query plans: scans vs index seeks; selectivity; statistics

### Indexing

- B-Tree / B+Tree, composite indexes, covering indexes
- Hash indexes (where applicable)
- Common pitfalls: wrong index order, low-selectivity columns

### Transactions & isolation

- ACID, write-ahead logging (WAL)
- Isolation: read committed, repeatable read, serializable
- Anomalies: dirty read, non-repeatable, phantom reads
- MVCC (common in Postgres/MySQL engines)

### NoSQL families

- Key-value, document, wide-column, graph, time-series
- Tradeoffs: schema flexibility vs guarantees

### Replication & sharding

- Primary/replica, async vs sync, failover
- Partitioning strategies: hash/range/consistent hashing
- Hot partitions, rebalancing

---

## 8) Distributed Systems (Everything is a tradeoff)

- Latency vs consistency vs availability
- **CAP**: partition tolerance is non-optional; choose C/A behavior under partition
- Consistency models: strong, eventual, causal
- Coordination: leader election, consensus (Raft/Paxos conceptually)
- Quorums, read/write repair
- Idempotency, deduplication keys
- Clock issues: drift, NTP, logical clocks (Lamport/vector)
- Failure modes: partial failure, split brain, cascading failures
- Patterns:
  - retries w/ backoff + jitter
  - circuit breaker, bulkhead, timeouts
  - saga vs 2PC
  - outbox pattern + CDC
- Exactly-once is usually “effectively once” with idempotency

---

## 9) Systems Design Patterns (Practical)

- Monolith vs modular monolith vs microservices
- API Gateway, BFF, service mesh (when/why)
- CQRS, event sourcing (where it helps, where it hurts)
- Async messaging: queues vs pub/sub vs streams
- Caching: CDN, reverse proxy, app cache, DB cache
- Rate limiting: token bucket, leaky bucket
- Search: inverted indexes (Elastic/OpenSearch)
- Storage: blob/object stores vs block storage

---

## 10) Security (You can’t skip this)

### Core concepts

- CIA triad, threat modeling
- AuthN vs AuthZ; least privilege
- Secrets management; key rotation

### Web security basics

- XSS (stored/reflected/DOM), CSP
- CSRF, SameSite cookies
- SQL injection, command injection
- SSRF, path traversal, deserialization
- Clickjacking, CORS misconceptions
- Session management, token storage tradeoffs

### Crypto (practical awareness)

- Hash vs HMAC vs encryption
- Symmetric (AES), asymmetric (RSA/ECC), signatures
- Password hashing: bcrypt/scrypt/argon2
- Don’t roll your own crypto

### Supply chain

- dependency auditing, lockfiles, SBOMs
- container image provenance

---

## 11) Compilers & Interpreters (Just enough)

- Lexer → parser → AST → semantic analysis → IR → optimization → codegen
- Type checking, symbol tables, scopes
- GC: mark/sweep, generational, stop-the-world vs incremental
- JIT basics: hot paths, inline caching

---

## 12) Computer Architecture (Performance intuition)

- CPU pipeline, branch prediction (don’t fight it)
- Caches (L1/L2/L3), locality, cache misses dominate
- SIMD/vectorization idea
- Disk vs RAM vs network orders of magnitude
- NUMA awareness in big systems

---

## 13) Concurrency & Parallelism

- Parallelism (do many) vs concurrency (manage many)
- Shared memory vs message passing
- Data races, deadlocks (4 Coffman conditions)
- Common models:
  - threads + locks
  - async event loop
  - actors/channels
- Patterns:
  - producer/consumer
  - work stealing
  - backpressure
- Observability: contention shows up as latency spikes

---

## 14) Testing & Verification

- Unit, integration, e2e, contract tests
- Property-based testing (invariants)
- Fuzzing (input chaos)
- Static analysis, linters, type systems as “cheap tests”
- Formal methods (when it matters): model checking, TLA+ awareness
- Test pyramid is context-dependent; reliability > ideology

---

## 15) Reliability Engineering (Production reality)

- SLO/SLI/SLA
- Error budgets
- Incident response: paging, triage, postmortems
- Observability:
  - logs, metrics, traces (and correlation IDs)
  - RED/USE/Golden signals
- Load testing, capacity planning
- Graceful degradation; feature flags
- Multi-region: latency vs complexity

---

## 16) Performance (End-to-end thinking)

- Measure first: profiling, flame graphs
- Latency budget: client + network + server + DB
- Caching strategy + invalidation (hard part)
- N+1 queries; batching; pagination
- Memory leaks, GC pauses
- Tail latency (p95/p99) matters more than average

---

## 17) Data / ML (CS-adjacent but common now)

- Data pipeline: ingest → clean → transform → serve
- Batch vs stream processing
- Feature engineering basics
- Model types: linear, trees, ensembles, deep nets (high-level)
- Evaluation: train/test split, leakage, metrics
- Serving: drift, monitoring, A/B testing

---

## 18) Human/Software Systems

- Requirements: functional vs non-functional
- Tradeoffs: build vs buy, simplicity vs flexibility
- Documentation: ADRs, runbooks
- API design: versioning, compatibility, deprecation
- Code review culture, technical debt management

---

## 19) “Must Know” Canonical Concepts List

- Big-O, recursion, DP
- Hashing + collisions
- Graph traversal
- ACID + isolation anomalies
- HTTP caching + TLS basics
- Event loop + async I/O
- Consistency tradeoffs in distributed systems
- Idempotency + retries
- Basic threat model + OWASP style issues
- Observability + SLO thinking

---

## 20) Quick Self-Assessment Checklist (Gaps Finder)

If any of these feel fuzzy, that’s your next study target:

- Explain **MVCC** and one isolation anomaly
- Explain **HTTP caching** with `ETag` + `Cache-Control`
- Explain **idempotency** and why retries are dangerous without it
- Explain **deadlock** and how to prevent it
- Explain **B-Tree** vs **LSM tree** at a high level
- Explain **CAP** with a real example
- Explain **DNS** resolution path + TTL caching
- Explain **XSS vs CSRF** and one mitigation each
- Explain **p99 latency** and why it matters

---

## 21) Recommended Learning Path (Minimal, High ROI)

1) DS&A fundamentals (to sharpen thinking)
2) OS + Networking (to reason about real performance bugs)
3) Databases + Transactions (to avoid production landmines)
4) Distributed systems (to design sane backends)
5) Security (to stop shipping vulnerabilities)
6) Reliability/observability (to keep it running)

---
