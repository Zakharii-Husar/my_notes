# Concurrency and Parallelism

Concurrency and parallelism are two of the most misused terms in software engineering — and confusing them leads to real architectural mistakes. This lesson clarifies the distinction, surveys the major models for writing concurrent code, covers the bugs that haunt concurrent systems, and presents practical patterns you'll encounter in production. If Lesson 1 covered the OS primitives, this lesson is about how to compose them into working systems.

---

## 1. Parallelism vs Concurrency

### 1.1 The Core Distinction

- **Concurrency** is about *managing* many things at once. It's a program structure — your code is organized to handle multiple tasks whose execution may overlap. A single-core CPU can run concurrent code by interleaving tasks.
- **Parallelism** is about *doing* many things at once. It requires multiple processing units (cores, CPUs, machines) executing simultaneously.

```
Concurrency (single core, interleaved):
  Task A: ██░░██░░██
  Task B: ░░██░░██░░

Parallelism (two cores, simultaneous):
  Core 1: ██████████  ← Task A
  Core 2: ██████████  ← Task B
```

A web server handling 10,000 connections on a single thread (via an event loop) is concurrent but not parallel. A matrix multiplication split across 8 CPU cores is parallel. Most real systems need both.

### 1.2 Why It Matters

Choosing incorrectly has consequences:
- Using threads for I/O-bound work → wasted memory and context-switch overhead
- Using a single event loop for CPU-bound work → blocks all other tasks
- Assuming parallelism when code is actually serialized by a lock → no speedup

**Amdahl's Law** quantifies the limit of parallelism. If 10% of your program is serial, the maximum speedup with infinite cores is 10x. Always identify the serial bottleneck first.

```
Speedup = 1 / (S + P/N)

  S = serial fraction
  P = parallelizable fraction (1 - S)
  N = number of processors

Example: S = 0.1, N = 8
  Speedup = 1 / (0.1 + 0.9/8) = 1 / 0.2125 ≈ 4.7x  (not 8x)
```

---

## 2. Communication Models

### 2.1 Shared Memory

Threads communicate by reading and writing shared variables. This is the default model in languages like C, C++, Java, and Python (with threads).

**Advantages**: Fast — no serialization or copying needed.
**Dangers**: Every piece of shared mutable state is a potential race condition. You must synchronize with locks, atomics, or careful design.

```python
# Shared memory: two threads increment a shared counter
import threading

counter = 0
lock = threading.Lock()

def worker():
    global counter
    for _ in range(100000):
        with lock:
            counter += 1

threads = [threading.Thread(target=worker) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)  # 400000 (correct only because of the lock)
```

### 2.2 Message Passing

Processes or actors communicate by sending immutable messages through channels, queues, or mailboxes. No shared state — each entity owns its data exclusively.

**Advantages**: Eliminates data races by design. Scales naturally across machines.
**Dangers**: Deadlock is still possible (A waits for B's reply, B waits for A's). Copying/serialization has overhead.

```go
// Message passing in Go with channels
func producer(ch chan<- int) {
    for i := 0; i < 100; i++ {
        ch <- i  // send
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for val := range ch {  // receive until closed
        fmt.Println(val)
    }
}

func main() {
    ch := make(chan int, 10)  // buffered channel
    go producer(ch)
    consumer(ch)
}
```

### 2.3 When to Use Which

| Factor | Shared Memory | Message Passing |
|---|---|---|
| Latency-critical | Preferred (no copy overhead) | Higher overhead |
| Safety-critical | Needs discipline | Safer by default |
| Distributed systems | Doesn't work across machines | Natural fit |
| Debugging | Harder (nondeterministic) | Easier (trace messages) |

In practice, most systems use both: shared memory within a process, message passing between processes or services.

---

## 3. Concurrency Bugs

### 3.1 Data Races

A data race occurs when two threads access the same memory location concurrently, at least one is a write, and there's no synchronization. Data races cause undefined behavior in C/C++ and subtle corruption everywhere else.

```c
// Data race: incrementing without synchronization
// Two threads running this simultaneously:
counter++;
// This compiles to: load → add → store
// Thread interleaving can lose updates:
//   T1: load counter (0)
//   T2: load counter (0)
//   T1: store counter (1)
//   T2: store counter (1)  ← lost T1's increment!
```

**Not all race conditions are data races.** A race condition is a logic bug where correctness depends on timing — it can exist even with proper synchronization (e.g., check-then-act patterns).

### 3.2 Deadlocks and the Four Coffman Conditions

Deadlock occurs when a set of threads are each waiting for a resource held by another thread in the set. All four **Coffman conditions** must hold simultaneously:

1. **Mutual exclusion**: Resources are held exclusively (not sharable).
2. **Hold and wait**: A thread holds one resource while waiting for another.
3. **No preemption**: Resources cannot be forcibly taken from a thread.
4. **Circular wait**: A cycle exists in the wait-for graph (A→B→C→A).

**Breaking any one condition prevents deadlock:**

| Condition | Prevention Strategy |
|---|---|
| Mutual exclusion | Use lock-free structures or read-write locks |
| Hold and wait | Acquire all locks atomically or release before requesting more |
| No preemption | Use `tryLock` with timeout; back off on failure |
| Circular wait | **Lock ordering** — always acquire locks in a global, consistent order |

Lock ordering is the most practical strategy:

```python
# Deadlock-prone:
def transfer_bad(account_a, account_b, amount):
    with account_a.lock:       # Thread 1: lock A, then B
        with account_b.lock:   # Thread 2: lock B, then A → deadlock
            account_a.balance -= amount
            account_b.balance += amount

# Deadlock-free: always lock the lower-ID account first
def transfer_safe(account_a, account_b, amount):
    first, second = sorted([account_a, account_b], key=lambda a: a.id)
    with first.lock:
        with second.lock:
            account_a.balance -= amount
            account_b.balance += amount
```

### 3.3 Starvation and Livelock

- **Starvation**: A thread can never make progress because other threads always take priority. Example: a writer starved by continuous readers on a read-write lock.
- **Livelock**: Threads keep responding to each other's actions but none makes progress. Example: two threads detect conflict and both back off, then both retry, detect conflict again, and repeat forever.

Fix livelock with **randomized backoff** — each thread waits a random interval before retrying.

---

## 4. Concurrency Models

### 4.1 Threads + Locks

The most direct model. You create threads and protect shared state with mutexes, condition variables, and RW locks.

**Pros**: Works everywhere, fine-grained control, good for CPU-bound parallelism.
**Cons**: Hard to reason about, easy to introduce deadlocks and races, doesn't scale well for high-concurrency I/O.

**Best for**: CPU-bound workloads, small numbers of threads, performance-critical systems.

### 4.2 Async / Event Loop

A single thread (or small pool) runs an event loop that dispatches callbacks or coroutines when I/O completes. Used by Node.js, Python asyncio, Rust's Tokio, Nginx.

```python
import asyncio

async def fetch(url):
    reader, writer = await asyncio.open_connection(url, 80)
    writer.write(b"GET / HTTP/1.0\r\nHost: " + url.encode() + b"\r\n\r\n")
    data = await reader.read(4096)  # yields control, no thread blocked
    writer.close()
    return data

async def main():
    # Fan out: 100 concurrent requests, ~1 thread
    results = await asyncio.gather(*[fetch("example.com") for _ in range(100)])
```

**Pros**: Handles thousands of I/O-bound tasks efficiently. No locking needed for single-threaded event loops.
**Cons**: CPU-bound work blocks the loop. "Colored function" problem — async is viral (all callers must be async). Debugging stack traces is harder.

**Best for**: Network servers, I/O-bound workloads, high-concurrency scenarios.

### 4.3 Actors / Channels

Each actor is an independent entity with its own state and a mailbox. Actors communicate by sending asynchronous messages. No shared state. Used by Erlang/OTP, Akka (Scala/Java), and inspired by Go's goroutines + channels.

```
Actor A                    Actor B
┌────────────┐  message   ┌────────────┐
│ state: 42  │ ────────→  │ state: 7   │
│ mailbox:[] │            │ mailbox:[m]│
└────────────┘  ←──────── └────────────┘
                 response
```

**Pros**: Excellent isolation, natural fit for distributed systems, fault tolerance via supervision trees (Erlang's "let it crash" philosophy).
**Cons**: Overhead of message serialization, harder to do shared-state operations efficiently, potential for mailbox overflow.

**Best for**: Distributed systems, fault-tolerant services, complex stateful workflows.

---

## 5. Practical Patterns

### 5.1 Producer/Consumer

One or more producers push work items into a bounded queue; one or more consumers pull and process them. The queue decouples production rate from consumption rate.

```
Producers         Queue (bounded)       Consumers
  P1  ──→  ┌───────────────────┐  ──→  C1
  P2  ──→  │ [item][item][item]│  ──→  C2
  P3  ──→  └───────────────────┘  ──→  C3
```

Key design decisions:
- **Bounded vs unbounded queue**: Bounded queues provide **backpressure** — producers block when the queue is full, preventing memory exhaustion.
- **Single vs multiple consumers**: Multiple consumers enable parallelism but require the work to be safely partitioned.

### 5.2 Work Stealing

Each worker thread has its own local deque (double-ended queue). When a worker's deque is empty, it "steals" tasks from the tail of another worker's deque.

```
Worker 1 deque:  [A][B][C]  ← push/pop from this end (LIFO, cache-friendly)
Worker 2 deque:  [D]
Worker 3 deque:  []         ← empty, steals C from Worker 1's tail
```

**Why it works well:**
- Workers mostly operate on their own local deque — no contention
- Stealing from the *opposite end* minimizes conflicts
- Naturally balances load across threads

Used by: Java ForkJoinPool, Rust's Rayon, Go's goroutine scheduler, Intel TBB.

### 5.3 Backpressure

Backpressure is a mechanism that slows down producers when consumers can't keep up. Without it, work piles up in memory and the system eventually crashes or degrades.

Ways to implement backpressure:
- **Bounded queues**: Producer blocks when queue is full
- **Rate limiting**: Accept only N requests per second
- **Load shedding**: Drop or reject excess work (return HTTP 503)
- **Credit-based flow control**: Consumer tells producer how much it can accept

```
Without backpressure:
  Producer: 1000 req/s → [queue grows unbounded] → Consumer: 100 req/s → OOM

With backpressure:
  Producer: 1000 req/s → [bounded queue, blocks at 100] → Consumer: 100 req/s → stable
```

Backpressure should be visible in your system: monitor queue depths and rejection rates.

---

## 6. Observability: Diagnosing Concurrency Issues

Concurrency problems often manifest as **latency spikes** or **throughput collapse** rather than outright crashes. Key signals:

- **Lock contention**: Threads spend time waiting on locks instead of doing work. Shows up as high CPU usage with low throughput, or as latency spikes at the 99th percentile.
- **Context switch rate**: Use `vmstat` or `pidstat` on Linux. High cs/s with degraded performance suggests too many threads.
- **Thread pool saturation**: All threads busy, queue growing. Tasks wait longer to start.

### Tools for Investigation

| Tool | What It Shows |
|---|---|
| `perf lock` | Lock contention statistics (Linux) |
| `top -H` | Per-thread CPU usage |
| `strace -f` | Syscalls across all threads |
| `go tool trace` | Goroutine scheduling, blocking (Go) |
| `async-profiler` | Lock/contention/wall-clock profiles (JVM) |
| Thread dump (jstack, SIGQUIT) | Where threads are blocked right now |

### Example Diagnosis

```
Symptom: API p99 latency spikes to 500ms every ~30s
Investigation:
  1. Thread dump shows 50 threads waiting on a single mutex
  2. Mutex protects a shared cache
  3. Cache has a 30s TTL — all threads stampede to refresh simultaneously

Fix: "Probabilistic early expiration" — each thread has a small random
     chance of refreshing before TTL expires, spreading the load.
```

---

## Exercises

1. **Parallelism ceiling**: Given a function where 25% of the work is inherently serial, calculate the maximum speedup with 2, 4, 8, and 16 cores using Amdahl's Law. At what point do additional cores stop helping?

2. **Producer/consumer implementation**: Implement a bounded producer/consumer queue in your language of choice. Use a mutex and two condition variables (one for "not full", one for "not empty"). Test with fast producers and slow consumers — verify that producers block when the queue is full.

3. **Deadlock lab**: Write a program with three threads and three locks that deadlocks. Then apply lock ordering to fix it. Bonus: implement a `tryLock`-based approach with timeout and exponential backoff.

4. **Event loop vs threads**: Write a simple HTTP client that fetches 100 URLs. Implement it twice — once with a thread pool (e.g., 10 threads) and once with async I/O (asyncio, Tokio, etc.). Compare total time, memory usage, and code complexity.

5. **Contention measurement**: Write a micro-benchmark with N threads all incrementing a shared atomic counter in a tight loop. Measure throughput as N increases from 1 to 16. Graph the results. Where does throughput plateau or decline? Why?

---

## Recommended Resources

- **"The Art of Multiprocessor Programming"** (Herlihy & Shavit) — deep dive into lock-free algorithms and concurrent data structures.
- **"Designing Data-Intensive Applications"** (Kleppmann), Chapter 7–9 — concurrency in distributed systems, transactions, consensus.
- **"Seven Concurrency Models in Seven Weeks"** (Butcher) — practical survey of threads, actors, CSP, data parallelism, and more.
- **Go Concurrency Patterns** (Pike, talks) — excellent introduction to CSP-style concurrency.
- **"Is Parallel Programming Hard?"** (McKenney) — free online, covers memory models and RCU in depth.
- **Erlang's "Learn You Some Erlang"** — actor model and fault tolerance in practice.
