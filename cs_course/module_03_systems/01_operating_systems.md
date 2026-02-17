# Operating Systems

An operating system is the layer of software between your application code and the hardware. Understanding how it manages processes, memory, I/O, and synchronization is essential — not just for systems programming, but for writing correct, performant software in any language. Bugs rooted in OS-level misunderstanding (memory leaks, race conditions, I/O bottlenecks) are among the hardest to diagnose. This lesson builds the mental model you need.

---

## 1. Processes and Threads

### 1.1 What Is a Process?

A process is an instance of a running program. The OS gives each process its own **isolated address space** — meaning one process cannot (normally) read or write another process's memory. This isolation is the foundation of system stability: a crash in one process doesn't take down the rest.

Each process has:
- A virtual address space (code, data, heap, stack)
- One or more threads of execution
- File descriptors, signal handlers, environment variables
- A process ID (PID) and a parent process ID (PPID)

### 1.2 Context Switching

When the OS switches the CPU from one process (or thread) to another, it performs a **context switch**:

1. Save the current process's register state (program counter, stack pointer, etc.)
2. Update the process control block (PCB)
3. Load the next process's saved state
4. Resume execution

Context switches are **expensive** — typically 1–10 microseconds, not counting cache disruption. Every switch potentially flushes warm CPU cache lines, causing subsequent cache misses. This is why minimizing unnecessary context switches matters for latency-sensitive applications.

### 1.3 Scheduling

The OS scheduler decides which process/thread runs next. Common strategies:

- **Round-robin**: Each process gets a fixed time slice (quantum). Fair but not optimal for all workloads.
- **Priority-based**: Higher-priority tasks preempt lower ones. Risk: **starvation** of low-priority tasks.
- **Completely Fair Scheduler (CFS)**: Linux's default. Uses a red-black tree to track virtual runtime and picks the task that has run the least.
- **Multi-level feedback queue (MLFQ)**: Adjusts priority based on observed behavior — CPU-bound tasks get demoted, I/O-bound tasks get promoted.

### 1.4 Threads vs Processes

| Aspect | Processes | Threads |
|---|---|---|
| Memory | Separate address spaces | Shared address space |
| Creation cost | Higher (fork + COW) | Lower |
| Communication | IPC (pipes, sockets, shared mem) | Direct memory sharing |
| Isolation | Strong | Weak — one thread can corrupt another |
| Failure impact | Contained | Can crash entire process |

**User-level threads** are managed by a runtime library (e.g., goroutines, green threads) — the kernel doesn't know about them. **Kernel threads** are scheduled by the OS directly. Most modern runtimes use an M:N model — M user-level threads multiplexed onto N kernel threads.

### 1.5 Concurrency Hazards

- **Race condition**: Two threads access shared data without proper synchronization; the outcome depends on timing.
- **Deadlock**: Two or more threads each hold a resource the other needs, and none can proceed.
- **Starvation**: A thread never gets the resources it needs because others keep taking them.
- **Livelock**: Threads keep changing state in response to each other but make no forward progress (imagine two people stepping aside for each other in a corridor).

```
Deadlock example (pseudocode):

Thread A:              Thread B:
  lock(mutex1)           lock(mutex2)
  lock(mutex2)  ← waits  lock(mutex1)  ← waits
  // both stuck forever
```

---

## 2. Memory

### 2.1 Virtual Memory and Paging

Each process sees a contiguous, private **virtual address space** (e.g., 0x0 to 2^48 on x86-64). The **Memory Management Unit (MMU)** translates virtual addresses to physical addresses using **page tables**.

Memory is divided into fixed-size **pages** (typically 4 KB). Benefits:
- Processes are isolated without relocating physical memory
- The OS can overcommit — allocate more virtual memory than physical RAM
- Unused pages can be swapped to disk

A **page fault** occurs when a process accesses a page not currently in physical memory. The OS then:
1. Traps to the kernel
2. Finds or allocates a physical frame
3. Loads the page from disk (if swapped) or zeros it (if new)
4. Updates the page table and resumes the process

**Minor** page faults (page is in memory but not mapped) are fast. **Major** page faults (requires disk I/O) can take milliseconds — orders of magnitude slower.

### 2.2 Address Spaces and Shared Memory

A typical process address space layout (high to low):

```
┌──────────────┐  High addresses
│    Stack      │  ← grows downward (local vars, call frames)
│      ↓        │
│   (gap)       │
│      ↑        │
│    Heap       │  ← grows upward (malloc / new)
│──────────────│
│    BSS        │  ← uninitialized globals (zeroed)
│    Data       │  ← initialized globals
│    Text       │  ← executable code (read-only)
└──────────────┘  Low addresses
```

**Shared memory** (`mmap`, `shmget`) lets multiple processes map the same physical pages into their address spaces — the fastest form of IPC, but requires synchronization.

### 2.3 CPU Caches and Locality

Modern CPUs have multi-level caches (L1 ~1ns, L2 ~4ns, L3 ~12ns, RAM ~100ns). Data moves in **cache lines** (typically 64 bytes). Two principles:

- **Temporal locality**: Recently accessed data is likely to be accessed again soon.
- **Spatial locality**: Data near recently accessed addresses is likely needed next.

This is why iterating an array is fast (sequential access fills cache lines efficiently) and chasing linked-list pointers is slow (random access defeats the prefetcher).

**False sharing** occurs when two threads modify different variables that happen to live on the same cache line. The cache coherence protocol forces the line to bounce between cores, destroying performance — even though the threads aren't logically sharing data.

```c
// False sharing example
struct CounterPair {
    long counter_a;  // Thread 1 writes this
    long counter_b;  // Thread 2 writes this
    // Both likely on the same 64-byte cache line!
};

// Fix: pad to separate cache lines
struct CounterPair {
    long counter_a;
    char _pad[56];   // push counter_b to the next cache line
    long counter_b;
};
```

### 2.4 Memory Allocation

**`malloc` / `new`** allocate heap memory. Under the hood, allocators like glibc's ptmalloc, jemalloc, or tcmalloc manage free lists, arenas, and size classes.

**Fragmentation**:
- **External fragmentation**: Free memory exists but is scattered in small non-contiguous blocks.
- **Internal fragmentation**: Allocated blocks are larger than requested (due to alignment/size classes).

**Garbage collection (GC) vs manual management**:

| GC (Java, Go, Python) | Manual (C, C++, Rust) |
|---|---|
| Safer — no use-after-free | Full control over allocation lifetime |
| GC pauses can hurt latency | Memory leaks if you forget to free |
| Higher memory overhead | Bugs: dangling pointers, double-free |
| Simpler developer experience | Rust's borrow checker: safety without GC |

---

## 3. I/O

### 3.1 Syscalls and File Descriptors

Applications talk to the OS via **system calls** (`open`, `read`, `write`, `close`, etc.). Each open file, socket, or pipe is represented by a **file descriptor** (a small integer). Three are pre-opened: `0` = stdin, `1` = stdout, `2` = stderr.

Syscalls are expensive (trap to kernel mode, context save/restore). User-space **buffering** (like C's `stdio` or Python's `io.BufferedReader`) batches small reads/writes into larger syscalls:

```c
// Without buffering: 1000 syscalls
for (int i = 0; i < 1000; i++)
    write(fd, &byte, 1);

// With buffering: ~1 syscall (flushed when buffer fills or on fclose)
for (int i = 0; i < 1000; i++)
    fputc(byte, file);
```

### 3.2 Blocking, Nonblocking, and Async I/O

- **Blocking I/O**: `read()` suspends the thread until data is available. Simple but ties up one thread per connection.
- **Nonblocking I/O**: `read()` returns immediately with `EAGAIN` if no data is ready. You poll repeatedly — wasteful on its own.
- **I/O multiplexing**: `select`, `poll`, `epoll` (Linux), `kqueue` (BSD/macOS) — wait for events on many file descriptors at once. This is the backbone of event-driven servers (Nginx, Node.js, Tokio).
- **Async I/O**: `io_uring` (Linux) — submit batches of I/O operations and get notified on completion, minimizing syscalls.

```
Scaling comparison:
  Blocking:       1 thread per connection → 10K threads = too many context switches
  epoll/kqueue:   1 thread handles 10K+ connections via event loop
```

### 3.3 Filesystems

A filesystem organizes data on disk. Key concepts:

- **Inodes**: Metadata structures that store file attributes (size, permissions, timestamps) and pointers to data blocks. Directories are files that map names to inode numbers.
- **Journaling** (ext4, NTFS): Write changes to a journal/log before committing to the main filesystem. On crash, replay the journal to maintain consistency.
- **Permissions**: Unix model — owner/group/others × read/write/execute. Check with `ls -l`, modify with `chmod`.

```bash
$ ls -l myfile.txt
-rw-r--r-- 1 zakharii dev 4096 Feb 17 10:00 myfile.txt
│├┤├┤├┤
│ │  │  └── others: read only
│ │  └── group: read only
│ └── owner: read + write
└── regular file (d = directory, l = symlink)
```

---

## 4. Synchronization Primitives

### 4.1 Mutex (Mutual Exclusion Lock)

Only one thread can hold a mutex at a time. All others block until it's released.

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100000):
        with lock:          # acquire → critical section → release
            counter += 1
```

**Danger**: Forgetting to release (crash inside critical section) → deadlock. Use RAII/context managers.

### 4.2 Read-Write Lock (RWLock)

Allows multiple concurrent readers OR one exclusive writer. Ideal when reads vastly outnumber writes.

```
Readers:  R R R R  (all run concurrently)
Writer:   W        (blocks until all readers finish; blocks new readers)
```

**Watch out**: Writer starvation if readers keep arriving.

### 4.3 Semaphore

A counter that allows up to N threads into a section simultaneously. A mutex is a semaphore with N=1.

Use case: Limiting concurrent database connections to a pool of 10:

```python
pool = threading.Semaphore(10)

def query_db():
    with pool:  # blocks if 10 threads are already inside
        execute_query()
```

### 4.4 Condition Variable

Lets a thread wait for a specific condition, then be woken by another thread. Always used with a mutex.

```python
cv = threading.Condition()
queue = []

def producer():
    with cv:
        queue.append(item)
        cv.notify()  # wake one waiting consumer

def consumer():
    with cv:
        while not queue:
            cv.wait()  # releases lock, sleeps, re-acquires on wake
        item = queue.pop(0)
```

### 4.5 Atomics and CAS

**Atomic operations** execute as indivisible units without needing a lock. The key primitive is **Compare-And-Swap (CAS)**:

```
CAS(address, expected, new_value):
    atomically:
        if *address == expected:
            *address = new_value
            return true
        return false
```

CAS is the building block for lock-free data structures. It can fail under contention (another thread changed the value), so it's typically used in a retry loop.

**Memory ordering** controls how operations become visible to other threads. High-level awareness:
- **Relaxed**: No ordering guarantees; fastest.
- **Acquire/Release**: Establishes a happens-before relationship between threads.
- **Sequentially consistent**: Strongest; all threads see operations in the same total order; slowest.

Most developers should use mutexes and let the library handle memory ordering. Reach for atomics only when profiling proves lock contention is a bottleneck.

---

## Exercises

1. **Process vs Thread**: Write a program that creates 4 child processes (each incrementing a shared counter via shared memory) and another version using 4 threads. Compare the code complexity and final counter value (with and without synchronization).

2. **Page fault observation**: On Linux, run `/usr/bin/time -v ./your_program` and observe "Major (requiring I/O) page faults" vs "Minor (reclaiming a frame) page faults". Allocate a large array and access it sequentially vs randomly — compare page fault counts.

3. **False sharing**: Write a benchmark where two threads increment separate counters in the same struct. Measure throughput, then add cache-line padding and measure again. Expect 2–10x improvement.

4. **I/O multiplexing**: Write a simple echo server using `select` or `epoll` that handles multiple simultaneous connections on a single thread.

5. **Deadlock creation and detection**: Write a program that intentionally deadlocks (two threads, two mutexes, opposite acquisition order). Then fix it by enforcing a consistent lock ordering.

---

## Recommended Resources

- **"Operating Systems: Three Easy Pieces"** (Arpaci-Dusseau) — free online at [ostep.org](https://pages.cs.wisc.edu/~remzi/OSTEP/). The best introduction to OS concepts.
- **"Linux Programming Interface"** (Kerrisk) — comprehensive reference for Linux syscalls and internals.
- **"Computer Systems: A Programmer's Perspective"** (Bryant & O'Hallaron) — excellent coverage of memory hierarchy, linking, and I/O.
- **Julia Evans' zines** — approachable visual explanations of syscalls, memory, and networking.
- **`strace` / `perf` / `valgrind`** — hands-on tools to observe syscalls, cache misses, and memory errors in your own programs.
