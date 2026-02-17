# Computer Architecture — Performance Intuition

Understanding computer architecture isn't about designing CPUs. It's about developing an intuition for *why* your code is fast or slow in ways that algorithm analysis alone can't explain. Two algorithms with the same Big-O complexity can differ by 100x in practice because one is cache-friendly and the other isn't. This lesson gives you the mental model to reason about those differences.

---

## 1. CPU Pipeline and Branch Prediction

### Why it matters

Modern CPUs don't execute one instruction at a time. They overlap many instructions in a **pipeline** — while one instruction is being executed, the next is being decoded, and the one after that is being fetched. This is how a CPU can sustain billions of operations per second. But pipelines have a vulnerability: **branches** (if/else, loops) disrupt the flow.

### How the pipeline works

A simplified pipeline has stages: **Fetch → Decode → Execute → Memory → Write-back**. Each stage handles a different instruction simultaneously, like an assembly line. When everything flows smoothly, the CPU completes one instruction per clock cycle despite each instruction taking multiple cycles to process.

### Branch prediction

When the CPU encounters a branch (`if` statement), it doesn't know which path to take until the condition is evaluated — which might be several pipeline stages away. Instead of stalling, the CPU **predicts** the branch outcome and speculatively executes that path. If the prediction is correct, no time is lost. If wrong, the CPU must **flush the pipeline** and restart — wasting 10-20 cycles.

Modern branch predictors are remarkably good (95-99% accuracy) for predictable patterns. They struggle with random or data-dependent branches.

### Practical implications

```c
// Branch-friendly: sorted data has predictable branches
// Processing sorted vs unsorted array can differ by 5-6x
for (int i = 0; i < size; i++) {
    if (data[i] >= 128)  // Predictable after sorting
        sum += data[i];
}
```

- **Don't fight the branch predictor** — keep branches predictable. Sorting data before processing can make code dramatically faster.
- **Branchless code** — sometimes you can replace branches with arithmetic: `sum += (data[i] >= 128) * data[i]`. Compilers often do this for you (CMOV instructions).
- **Virtual dispatch** (vtable calls, polymorphism) is a form of indirect branch that's harder to predict. This is one reason why tight loops over polymorphic objects can be slow.

---

## 2. Caches (L1/L2/L3) and Memory Hierarchy

### Why it matters

The gap between CPU speed and main memory (RAM) speed is enormous. A CPU can execute an instruction in ~0.3 ns, but fetching from RAM takes ~100 ns — a 300x difference. Caches exist to bridge this gap by keeping frequently accessed data close to the CPU. **Cache misses dominate performance** in most real-world programs.

### The memory hierarchy

| Level      | Typical Size | Latency       | Analogy           |
|------------|-------------|---------------|-------------------|
| Registers  | ~1 KB       | ~0.3 ns       | Your hands        |
| L1 Cache   | 32-64 KB    | ~1 ns         | Your desk         |
| L2 Cache   | 256 KB-1 MB | ~4-7 ns       | Your bookshelf    |
| L3 Cache   | 4-64 MB     | ~10-30 ns     | The filing cabinet|
| RAM        | 8-512 GB    | ~80-100 ns    | The library       |
| SSD        | 256 GB-4 TB | ~10-100 μs    | Another building  |
| HDD        | 1-20 TB     | ~5-10 ms      | Another city      |
| Network    | Unlimited   | ~0.5-150 ms   | Another country   |

Each level is roughly 3-10x slower than the one above. The CPU doesn't fetch individual bytes — it fetches **cache lines** (typically 64 bytes). When you access one byte, the surrounding 63 come along for free.

### Locality

- **Temporal locality** — if you accessed data recently, you'll likely access it again soon. Loop variables, counters, and hot data structures exploit this.
- **Spatial locality** — if you accessed address X, you'll likely access X+1, X+2, etc. This is why arrays are cache-friendly and linked lists are not.

### Cache-hostile vs cache-friendly code

```c
// Cache-friendly: sequential access (spatial locality)
// Accesses memory in order: arr[0], arr[1], arr[2], ...
int sum = 0;
for (int i = 0; i < N; i++)
    sum += arr[i];

// Cache-hostile: random access pattern
// Each access likely causes a cache miss
int sum = 0;
for (int i = 0; i < N; i++)
    sum += arr[random_indices[i]];
```

**Row-major vs column-major traversal** is the classic example. A 2D array stored row-major accessed column-first causes a cache miss on nearly every access:

```c
// Fast: row-major traversal (C stores arrays row-major)
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        process(matrix[i][j]);

// Slow: column-major traversal — cache miss every iteration
for (int j = 0; j < cols; j++)
    for (int i = 0; i < rows; i++)
        process(matrix[i][j]);
```

For large matrices, the difference can be 10-50x.

### Practical implications

- **Prefer arrays/vectors over linked lists** — contiguous memory is cache-friendly
- **Keep hot data small** — a struct that fits in one cache line is faster than one spanning two
- **Avoid pointer chasing** — following pointers (`node->next->next`) often means cache misses
- **Batch similar operations** — process all items of one type before switching to another (data-oriented design)

---

## 3. SIMD / Vectorization

### Why it matters

**SIMD (Single Instruction, Multiple Data)** allows one CPU instruction to operate on multiple data elements simultaneously. Instead of adding two numbers, you add 4, 8, or even 16 pairs at once. Modern CPUs have wide SIMD registers (128-512 bits) that can dramatically speed up data-parallel workloads.

### How it works

```
// Scalar (one at a time):       // SIMD (four at once):
a[0] + b[0] → c[0]              a[0..3] + b[0..3] → c[0..3]
a[1] + b[1] → c[1]              (single instruction)
a[2] + b[2] → c[2]
a[3] + b[3] → c[3]
```

### Auto-vectorization

Modern compilers (GCC, Clang, MSVC) can automatically vectorize simple loops — but they need help:

- Data must be in contiguous arrays (not linked lists)
- Loop body should be simple and uniform (no complex branches)
- No loop-carried dependencies (iteration `i` shouldn't depend on `i-1`)
- Data types should be uniform (all `float`, all `int32`, etc.)

### Practical implications

- **You rarely write SIMD intrinsics** — the compiler does it if your code is structured well
- **Data layout matters** — arrays of structs (AoS) vs structs of arrays (SoA). SoA is more SIMD-friendly:

```
// AoS: Hard to vectorize — x, y, z interleaved in memory
struct Point { float x, y, z; };
Point points[1000];

// SoA: Easy to vectorize — all x values contiguous
struct Points { float x[1000]; float y[1000]; float z[1000]; };
```

- **Libraries leverage SIMD for you** — NumPy, image processing libraries, JSON parsers (simdjson) use SIMD internally. Choosing the right library is often the easiest way to benefit.

---

## 4. Orders of Magnitude: Disk vs RAM vs Network

### Why it matters

One of the most important skills in systems design is knowing the rough cost of operations. If you think disk access is "kind of slow" but don't know it's 100,000x slower than L1 cache, you'll make bad architectural decisions.

### The numbers you should know

| Operation                        | Time         |
|----------------------------------|-------------|
| L1 cache reference               | 1 ns        |
| L2 cache reference               | 4 ns        |
| Branch mispredict                | 5 ns        |
| Mutex lock/unlock                | 25 ns       |
| Main memory reference            | 100 ns      |
| Compress 1KB with Snappy         | 3 μs        |
| Read 1 MB sequentially from RAM  | 3-10 μs     |
| SSD random read (4KB)            | 16 μs       |
| Read 1 MB sequentially from SSD  | 50-200 μs   |
| Round trip within datacenter     | 500 μs      |
| Read 1 MB sequentially from HDD  | 2-5 ms      |
| Disk seek (HDD)                  | 5-10 ms     |
| CA to Netherlands round trip     | 150 ms      |

*(These are approximate and vary by hardware generation, but the ratios are what matter.)*

### Key takeaways

- **RAM is ~1000x faster than SSD for random access**. This is why databases keep indexes in memory.
- **Sequential access is dramatically faster than random** for all storage media. This is why log-structured storage (LSM trees, Kafka) writes sequentially.
- **Network within a datacenter (~500 μs) is comparable to SSD access**. This is why distributed caching (Redis) works — network to cache is faster than disk.
- **Cross-region network (~150 ms) dominates everything**. No amount of code optimization matters if you're making unnecessary cross-region calls.

---

## 5. NUMA Awareness

### Why it matters

On multi-socket servers (common in databases, big data systems), the memory architecture is **NUMA (Non-Uniform Memory Access)**. Each CPU socket has its own local memory, and accessing another socket's memory is 2-3x slower. On a 2-socket server, half your memory is "far" memory.

### How it works

```
┌────────────────┐     ┌────────────────┐
│   CPU Socket 0 │     │   CPU Socket 1 │
│   ┌──────────┐ │     │ ┌──────────┐   │
│   │  Cores   │ │     │ │  Cores   │   │
│   └──────────┘ │     │ └──────────┘   │
│   ┌──────────┐ │     │ ┌──────────┐   │
│   │ Local RAM│ │◄───►│ │ Local RAM│   │
│   │ (fast)   │ │ QPI │ │ (fast)   │   │
│   └──────────┘ │     │ └──────────┘   │
└────────────────┘     └────────────────┘

Accessing local RAM: ~100 ns
Accessing remote RAM: ~200-300 ns (through interconnect)
```

### Practical implications

- **Most developers don't need to worry about NUMA** — it matters for databases (PostgreSQL, MySQL), JVMs under heavy load, and high-performance computing.
- **Operating systems try to allocate memory near the requesting CPU** — but under memory pressure, this can fail.
- **NUMA-unaware applications** can lose 30-50% performance on multi-socket systems. If your database is slower than expected on a big server, NUMA is a common culprit.
- **Cloud VMs** usually abstract NUMA away, but very large instances may span sockets.
- **Key tool**: `numactl` on Linux lets you pin processes to specific NUMA nodes and see memory topology.

---

## Exercises

1. **Pipeline**: Explain why a loop that processes sorted data with an `if` condition can be faster than the same loop over unsorted data, even though the work per element is identical.
2. **Cache**: You have two implementations of a 1000x1000 matrix multiply. One iterates `(i, j, k)` and the other `(i, k, j)`. Which is faster and why? (Hint: think about which array is accessed sequentially in the inner loop.)
3. **Orders of magnitude**: Your web API makes 3 database queries (each ~2ms), 1 Redis call (~0.5ms), and does ~50μs of computation per request. Draw a latency breakdown. Where would you optimize first? What if you added a cross-region API call?
4. **Cache lines**: A struct has fields `{int id; char name[128]; bool active;}`. You frequently scan an array of these structs checking only the `active` field. Why is this cache-inefficient? How would you restructure the data?
5. **SIMD**: Given a function that computes the dot product of two float arrays, what properties make it a good candidate for auto-vectorization? What would prevent the compiler from vectorizing it?

---

## Recommended Resources

- **Article**: "Latency Numbers Every Programmer Should Know" by Jeff Dean (and the interactive version by Colin Scott) — bookmark this
- **Article**: "What Every Programmer Should Know About Memory" by Ulrich Drepper — the definitive deep dive on memory hierarchy
- **Book**: *Computer Systems: A Programmer's Perspective* (CS:APP) by Bryant & O'Hallaron — the best systems book for working programmers
- **Book**: *Computer Architecture: A Quantitative Approach* by Hennessy & Patterson — the classic, if you want to go deeper
- **Talk**: "Performance Matters" by Emery Berger (Strange Loop) — great examples of how architecture affects real code
- **Tool**: `perf stat` on Linux — run `perf stat ./your_program` to see cache misses, branch mispredictions, and IPC in your own code
- **Tool**: Compiler Explorer (godbolt.org) — see what your compiler actually generates, including SIMD instructions

---

*Previous lesson: [Discrete Math](./01_discrete_math.md) | Next lesson: [Programming Language Concepts](./03_programming_language_concepts.md)*
