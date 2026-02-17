# Distributed Systems

The moment your system runs on more than one machine, everything gets harder. Networks are unreliable, clocks disagree, processes crash independently, and you can no longer pretend that a function call will definitely succeed. Distributed systems engineering is the discipline of building reliable, correct behavior on top of unreliable components — and understanding which guarantees you're actually getting. These concepts aren't academic curiosities; they show up every time you use a database replica, a message queue, a CDN, or a multi-region deployment.

---

## The Fundamental Trade-offs

### Latency vs Consistency vs Availability

Every distributed system is constantly trading between three things:

- **Latency**: how fast a request completes. Synchronous replication to three data centers is consistent but slow
- **Consistency**: whether all nodes see the same data at the same time
- **Availability**: whether the system responds to every request (even if the answer might be stale)

You can't maximize all three. The engineering challenge is choosing which one to relax, and when.

### CAP Theorem

CAP states that a distributed system can provide at most two of three guarantees: **Consistency**, **Availability**, and **Partition tolerance**.

The critical insight: **partition tolerance is not optional**. Networks *will* partition — switches fail, cables get cut, cloud availability zones lose connectivity. The real question is:

> When a network partition happens, do you choose Consistency (refuse to serve potentially stale data) or Availability (keep serving, accept stale reads)?

- **CP systems** (e.g., ZooKeeper, etcd): during a partition, nodes that can't reach a quorum stop serving. You get correct answers or no answers
- **AP systems** (e.g., Cassandra, DynamoDB in default mode): during a partition, all nodes keep serving. You get answers, but they might be stale or conflicting

In practice, most systems aren't purely CP or AP — they make different choices for different operations. A system might be CP for writes and AP for reads.

### PACELC: Beyond CAP

CAP only describes behavior during partitions. PACELC extends it: when there's a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose **L**atency or **C**onsistency.

This captures the real-world truth: even when everything is working, you're still trading latency for consistency.

---

## Consistency Models

"Consistent" means different things in different contexts. These models describe what a reader can observe.

### Strong Consistency (Linearizability)

Every read returns the most recent write. The system behaves as if there's a single copy of the data. This is what you intuitively expect but it's the most expensive to provide.

- **Example**: after a bank transfer completes, both accounts immediately reflect the new balances from any node
- **Cost**: requires coordination (consensus) on every write, increasing latency
- **Who provides it**: Spanner (using TrueTime), CockroachDB, single-leader databases for single-key operations

### Eventual Consistency

If no new updates are made, all replicas will *eventually* converge to the same value. No guarantee about when, and reads might return stale data in the meantime.

- **Example**: DNS propagation — you update a record, and it takes minutes/hours for all resolvers to see it
- **Useful when**: staleness is tolerable (social media likes count, product view counters)
- **Danger**: "eventually" has no time bound. Code that assumes consistency will have subtle bugs

### Causal Consistency

If operation A *causally depends* on operation B, then everyone sees B before A. Operations with no causal relationship can be seen in any order.

- **Example**: if user posts a message and then edits it, no one should see the edit without the original message
- **Middle ground**: cheaper than strong consistency, stronger guarantees than eventual
- **Implementation**: vector clocks or explicit dependency tracking

```
Timeline example:

  Strong:     Write X=5 ──→ Any read returns 5 immediately
  Causal:     Write X=5 ──→ Read might return old value unless causally related
  Eventual:   Write X=5 ──→ Read returns old value for some time, eventually 5
```

---

## Coordination and Consensus

When multiple nodes need to agree on something (who is the leader? did this transaction commit?), you need consensus.

### Leader Election

Many systems use a single leader to serialize writes. But what happens when the leader dies?

- Remaining nodes must agree on a new leader
- During election, the system may be unavailable for writes
- **Split brain risk**: if the old leader isn't actually dead (just slow), you can have two leaders accepting conflicting writes
- **Solutions**: fencing tokens — each leader gets a monotonically increasing token; storage nodes reject writes from old tokens

### Consensus Algorithms

**Raft** and **Paxos** solve the consensus problem: getting a cluster of nodes to agree on a sequence of values even when some nodes fail.

**Raft** (easier to understand):
1. One node is elected leader
2. Leader receives all writes and replicates them to followers
3. A write is committed once a **majority** (quorum) acknowledges it
4. If the leader fails, followers with the most up-to-date logs can become the new leader
5. Terms (epochs) prevent stale leaders from causing confusion

**Paxos** (historically important, notoriously hard to implement correctly):
- Solves the same problem with proposers, acceptors, and learners
- Multi-Paxos optimizes for sequences of decisions
- Most production systems use Raft or a Paxos variant (e.g., Google's Chubby)

**Key takeaway**: consensus requires a majority of nodes to be reachable. A 5-node cluster tolerates 2 failures, a 3-node cluster tolerates 1.

### Quorums

A quorum-based system doesn't require all nodes to participate — just enough:

- **Write quorum (W)**: number of nodes that must acknowledge a write
- **Read quorum (R)**: number of nodes that must respond to a read
- **Rule**: if `W + R > N` (total nodes), reads and writes overlap, so reads see the latest write

```
N=3 replicas:

  W=2, R=2:  Strong consistency (overlap guaranteed)
  W=1, R=1:  Fast but eventual consistency (no overlap)
  W=3, R=1:  Write-heavy penalty, cheap reads
```

**Read repair**: during a quorum read, if a node returns stale data, the coordinator sends it the latest version. This heals inconsistencies passively.

**Write repair (anti-entropy)**: background process that compares replicas and fixes divergence using Merkle trees.

---

## Time and Clocks

Distributed systems have no shared global clock. This causes more bugs than you'd expect.

### Clock Drift and NTP

- Physical clocks on different machines drift apart (crystal oscillators are imperfect)
- **NTP** (Network Time Protocol) synchronizes clocks, but only to within milliseconds — sometimes tens of milliseconds
- **Consequence**: you cannot use wall-clock timestamps to order events across machines

```
Machine A clock: 14:00:00.000
Machine B clock: 14:00:00.050  (50ms ahead)

Event on A at 14:00:00.030 (A's clock)
Event on B at 14:00:00.040 (B's clock)

Did A happen before B? A's wall clock says yes,
but B's clock is 50ms fast, so B actually happened at 13:59:59.990.
B happened BEFORE A. The timestamps lied.
```

### Logical Clocks

Since physical clocks can't reliably order events, we use logical clocks:

**Lamport Clocks**:
- Each node maintains a counter
- On local event: increment counter
- On send: attach counter to message
- On receive: set counter to `max(local, received) + 1`
- Gives a *partial* ordering: if `L(A) < L(B)`, A *might* have happened before B. But `L(A) < L(B)` doesn't prove causality

**Vector Clocks**:
- Each node maintains a vector of counters (one per node)
- Captures true causal relationships: if `VC(A) < VC(B)` (every element of A ≤ corresponding element of B, and at least one is strictly less), then A causally precedes B
- If neither VC(A) < VC(B) nor VC(B) < VC(A), the events are concurrent (no causal relationship)
- **Trade-off**: vector size grows with number of nodes

---

## Failure Modes

In a distributed system, failures are partial and unpredictable.

### Partial Failure

Some nodes work while others don't. A request might succeed on 2 of 3 replicas. The network might deliver a message but lose the acknowledgment. You can't always tell the difference between "the node is slow" and "the node is dead."

### Split Brain

A network partition divides the cluster into two groups, each believing the other is dead. Both sides might elect a leader and accept writes, creating conflicting state that's painful to merge after the partition heals.

**Mitigations**: quorum-based systems (only the majority partition can make progress), fencing tokens, STONITH ("shoot the other node in the head" — forcibly power off the questionable node).

### Cascading Failures

One component fails, causing others to overload as they absorb its traffic, which causes them to fail too — a domino effect.

- **Example**: one database replica goes down → traffic shifts to remaining replicas → they can't handle the load → they fail too → total outage
- **Classic trigger**: retry storms — every client retries simultaneously, amplifying the load

---

## Resilience Patterns

### Retries with Exponential Backoff and Jitter

Retrying is essential, but naive retries cause thundering herds:

```python
import random
import time

def retry_with_backoff(func, max_retries=5, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            # Exponential backoff with full jitter
            delay = base_delay * (2 ** attempt)
            jittered_delay = random.uniform(0, delay)
            time.sleep(jittered_delay)
```

- **Exponential backoff**: wait 1s, 2s, 4s, 8s... gives the system time to recover
- **Jitter**: randomize the delay so retries from many clients don't all hit at the same moment
- **Cap the maximum delay**: don't wait forever

### Circuit Breaker

Stop sending requests to a failing service so it can recover:

```
States:
  CLOSED  ──(failures exceed threshold)──→  OPEN
  OPEN    ──(timeout expires)──→             HALF-OPEN
  HALF-OPEN ──(test request succeeds)──→     CLOSED
  HALF-OPEN ──(test request fails)──→        OPEN
```

- **Closed**: requests flow normally, failures are counted
- **Open**: all requests immediately fail (fast-fail) without calling the downstream service
- **Half-open**: allow one test request through. If it succeeds, close the circuit; if it fails, re-open

### Bulkhead

Isolate components so one failure doesn't consume all resources:

- Separate thread pools per downstream service
- If service A is slow, only its thread pool fills up — services B and C are unaffected
- Named after ship compartments that prevent a single hull breach from sinking the whole ship

### Timeouts

Every remote call must have a timeout. Without one, a hanging downstream service will tie up threads/connections indefinitely.

- Set timeouts based on observed p99 latency, not average
- Distinguish between **connection timeout** (can't establish connection) and **read timeout** (connected but waiting for response)

---

## Idempotency and Exactly-Once Semantics

### The Problem

Networks can duplicate messages. A client sends a payment request, the server processes it, but the acknowledgment gets lost. The client retries — was the payment processed twice?

### Idempotency Keys

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

```python
# Client generates a unique idempotency key per logical operation
POST /payments
Idempotency-Key: txn_abc123
{
    "amount": 50.00,
    "to": "merchant_xyz"
}

# Server logic:
# 1. Check if txn_abc123 was already processed
# 2. If yes, return the stored result (don't re-process)
# 3. If no, process and store the result keyed by txn_abc123
```

### "Exactly-Once" is Really "Effectively Once"

True exactly-once delivery is impossible in a distributed system (proven by the Two Generals' Problem). What we actually achieve:

- **At-most-once**: send and forget. Message might be lost
- **At-least-once**: retry until acknowledged. Message might be duplicated
- **Effectively once**: at-least-once delivery + idempotent processing = the *effect* happens exactly once

### Deduplication Keys

Similar to idempotency keys but used in message processing:

- Producer attaches a unique ID to each message
- Consumer tracks which IDs it has processed (in a deduplication store)
- On receiving a message, check the ID — if already processed, skip it

---

## Distributed Transaction Patterns

### Two-Phase Commit (2PC)

A coordinator ensures all participants either commit or abort:

1. **Prepare phase**: coordinator asks all participants "can you commit?" Each responds yes/no
2. **Commit phase**: if all said yes, coordinator sends "commit." If any said no, coordinator sends "abort"

**Problems**: the coordinator is a single point of failure. If it crashes between phases, participants are stuck in an uncertain state ("in-doubt" transactions) holding locks.

### Saga Pattern

An alternative for long-running transactions across services. Each step has a compensating action:

```
Book Flight → Book Hotel → Charge Payment
    ↓              ↓             ↓
Cancel Flight  Cancel Hotel   Refund Payment  (compensating actions)
```

- If "Charge Payment" fails, execute compensating actions in reverse: cancel hotel, cancel flight
- **Choreography**: each service listens for events and reacts (decentralized, harder to debug)
- **Orchestration**: a central coordinator tells each service what to do (easier to reason about, coordinator is a dependency)

### Outbox Pattern + Change Data Capture (CDC)

Problem: you need to update a database AND publish a message atomically. If you update the DB and then the message broker is down, the message is lost. If you send the message first and the DB write fails, you published a lie.

**Solution — Outbox Pattern**:

1. Write the business data AND an "outbox" event to the same database in a single transaction
2. A separate process reads the outbox table and publishes events to the message broker
3. After publishing, mark the outbox entry as sent

**CDC** (Change Data Capture) automates step 2 by tailing the database's transaction log (e.g., Debezium reading the PostgreSQL WAL) and publishing changes as events. This is reliable because the transaction log is the source of truth.

---

## Exercises

1. **CAP Analysis**: pick three real systems you use (e.g., PostgreSQL with streaming replication, Redis Cluster, S3). For each, describe what happens during a network partition. Which side of the CP/AP spectrum does each fall on, and for which operations?

2. **Clock Debugging**: two services each log timestamps for the same request. Service A logs `t=1000ms`, Service B logs `t=995ms`. Service A calls Service B. Is it possible that A's log entry was actually created *before* B's, despite the timestamp ordering? Explain using clock drift.

3. **Idempotency Design**: design an idempotent API for a money transfer endpoint. What key would you use? Where would you store processed keys? What happens if the key store itself is unavailable?

4. **Saga Design**: design a saga for an e-commerce checkout that involves: (a) reserving inventory, (b) charging payment, (c) creating a shipment. Define the compensating action for each step. What happens if the compensation itself fails?

5. **Retry Analysis**: a service has 3 replicas behind a load balancer. Each replica has a 5% chance of transient failure. If a client retries 3 times (no backoff), what's the probability that all 3 attempts fail? Now consider: if all 1,000 clients retry simultaneously, what happens to the replicas?

---

## Recommended Resources

- **Designing Data-Intensive Applications** by Martin Kleppmann — the definitive book on distributed systems for practitioners. Chapters 5 (Replication), 7 (Transactions), 8 (Trouble with Distributed Systems), and 9 (Consistency and Consensus) are essential
- **The Raft Paper**: "In Search of an Understandable Consensus Algorithm" by Ongaro & Ousterhout — Raft was explicitly designed to be understandable. Read the paper; it's one of the most accessible in CS
- **Jepsen** (jepsen.io) — Kyle Kingsbury's analyses of distributed databases under failure conditions. Eye-opening for understanding real-world consistency guarantees
- **AWS Builders' Library** — practical articles on retry strategies, circuit breakers, timeouts, and operational patterns from engineers running one of the largest distributed systems
- **"Time, Clocks, and the Ordering of Events in a Distributed System"** by Leslie Lamport (1978) — the foundational paper on logical clocks. Short and readable
- **The Fallacies of Distributed Computing** — Peter Deutsch's classic list of assumptions that developers new to distributed systems incorrectly make
