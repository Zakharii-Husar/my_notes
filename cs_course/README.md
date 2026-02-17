# Computer Science Course

A self-paced computer science course built from the [CS Cheatsheet](./CS_Cheatsheet.md). Each module expands the cheatsheet's bullet points into structured lessons with explanations, examples, exercises, and resources.

---

## Course Structure

The course follows a bottom-up learning path: foundations first, then progressively higher-level topics. Each module builds on the previous ones.

### Module 1: Foundations
> *The bedrock everything else is built on.*

- [Discrete Math](./module_01_foundations/01_discrete_math.md) — logic, proofs, sets, graphs, combinatorics
- [Computer Architecture](./module_01_foundations/02_computer_architecture.md) — CPU, caches, memory hierarchy, performance intuition
- [Programming Language Concepts](./module_01_foundations/03_programming_language_concepts.md) — types, memory models, polymorphism, error handling

### Module 2: Data Structures & Algorithms
> *The core toolkit for solving problems.*

- [Data Structures](./module_02_dsa/01_data_structures.md) — arrays, hash tables, trees, heaps, graphs, advanced structures
- [Algorithms & Complexity](./module_02_dsa/02_algorithms_and_complexity.md) — Big-O, design patterns, core problems, complexity theory

### Module 3: Systems
> *Understanding the machine your code runs on.*

- [Operating Systems](./module_03_systems/01_operating_systems.md) — processes, memory, I/O, synchronization
- [Concurrency & Parallelism](./module_03_systems/02_concurrency_and_parallelism.md) — threads, async, actors, patterns
- [Compilers & Interpreters](./module_03_systems/03_compilers_and_interpreters.md) — lexing, parsing, type checking, GC, JIT

### Module 4: Networking
> *How systems talk to each other.*

- [Networking](./module_04_networking/01_networking.md) — TCP/IP, HTTP, DNS, TLS, WebSocket

### Module 5: Databases
> *Where data lives and how to manage it.*

- [Databases](./module_05_databases/01_databases.md) — data modeling, SQL, indexing, transactions, NoSQL, replication

### Module 6: Distributed Systems
> *Scaling beyond a single machine.*

- [Distributed Systems](./module_06_distributed_systems/01_distributed_systems.md) — CAP, consistency models, consensus, failure modes
- [Systems Design Patterns](./module_06_distributed_systems/02_systems_design_patterns.md) — architecture styles, messaging, caching, rate limiting

### Module 7: Security
> *You can't skip this.*

- [Security](./module_07_security/01_security.md) — threat modeling, web security, crypto basics, supply chain

### Module 8: Production Engineering
> *Building it is half the job. Running it is the other half.*

- [Testing & Verification](./module_08_production/01_testing_and_verification.md) — test types, property-based testing, fuzzing, formal methods
- [Reliability Engineering](./module_08_production/02_reliability_engineering.md) — SLOs, observability, incident response, graceful degradation
- [Performance](./module_08_production/03_performance.md) — profiling, latency budgets, caching, tail latency

### Module 9: Data & ML
> *CS-adjacent but increasingly essential.*

- [Data & ML](./module_09_data_ml/01_data_and_ml.md) — pipelines, processing models, model basics, serving

### Module 10: Professional Practice
> *The human side of software.*

- [Human & Software Systems](./module_10_professional/01_human_software_systems.md) — requirements, tradeoffs, API design, technical debt

---

## How to Use This Course

1. **Follow the module order** — each module builds on prior knowledge
2. **Read the cheatsheet first** — use [CS_Cheatsheet.md](./CS_Cheatsheet.md) as a map and quick reference
3. **Do the exercises** — each lesson ends with exercises; don't skip them
4. **Track your progress** — use the checklist below
5. **Use the self-assessment** — Section 20 of the cheatsheet has gap-finder questions

---

## Progress Tracker

- [ ] **Module 1: Foundations**
  - [ ] Discrete Math
  - [ ] Computer Architecture
  - [ ] Programming Language Concepts
- [ ] **Module 2: Data Structures & Algorithms**
  - [ ] Data Structures
  - [ ] Algorithms & Complexity
- [ ] **Module 3: Systems**
  - [ ] Operating Systems
  - [ ] Concurrency & Parallelism
  - [ ] Compilers & Interpreters
- [ ] **Module 4: Networking**
  - [ ] Networking
- [ ] **Module 5: Databases**
  - [ ] Databases
- [ ] **Module 6: Distributed Systems**
  - [ ] Distributed Systems
  - [ ] Systems Design Patterns
- [ ] **Module 7: Security**
  - [ ] Security
- [ ] **Module 8: Production Engineering**
  - [ ] Testing & Verification
  - [ ] Reliability Engineering
  - [ ] Performance
- [ ] **Module 9: Data & ML**
  - [ ] Data & ML
- [ ] **Module 10: Professional Practice**
  - [ ] Human & Software Systems

---

## Self-Assessment (from the Cheatsheet)

Once you've completed the course, you should be able to confidently explain all of these:

- [ ] MVCC and one isolation anomaly
- [ ] HTTP caching with `ETag` + `Cache-Control`
- [ ] Idempotency and why retries are dangerous without it
- [ ] Deadlock and how to prevent it
- [ ] B-Tree vs LSM tree at a high level
- [ ] CAP with a real example
- [ ] DNS resolution path + TTL caching
- [ ] XSS vs CSRF and one mitigation each
- [ ] p99 latency and why it matters
