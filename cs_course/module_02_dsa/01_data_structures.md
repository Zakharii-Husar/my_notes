# Data Structures

Data structures are the containers that shape how programs store, access, and manipulate information. Choosing the right structure determines whether an operation takes microseconds or minutes, whether your program fits in memory or thrashes swap, and whether your code is simple or tangled. This lesson builds intuition for the most important structures — from everyday arrays and hash tables to specialized structures used in databases, analytics, and concurrent systems.

---

## Why Data Structures Matter

Every non-trivial program is a conversation between algorithms and data structures. You can have the cleverest algorithm in the world, but if you store your data in a linked list when you need random access, performance collapses. Conversely, a mediocre algorithm over a well-chosen structure often beats a brilliant algorithm over a poor one.

The key questions when choosing a structure:

- **What operations dominate?** Lookups? Insertions? Ordered traversal? Range queries?
- **What are the size characteristics?** Dozens of items? Millions? Billions?
- **What are the concurrency requirements?** Single-threaded? Multi-reader? Multi-writer?
- **What are the durability requirements?** In-memory only? Persisted to disk?

---

## Arrays and Dynamic Arrays

### Core Concept

An **array** is a contiguous block of memory holding elements of the same size. Because elements sit next to each other, the CPU can calculate any element's address with simple arithmetic: `base + index * element_size`. This gives O(1) random access.

A **dynamic array** (Python `list`, Java `ArrayList`, C++ `std::vector`) wraps a fixed array and grows automatically. When it runs out of capacity, it allocates a new array (typically 2x the size) and copies everything over.

### Key Properties

| Operation | Array (fixed) | Dynamic Array |
|---|---|---|
| Access by index | O(1) | O(1) |
| Append | N/A | O(1) amortized |
| Insert at position i | O(n) | O(n) |
| Delete at position i | O(n) | O(n) |
| Search (unsorted) | O(n) | O(n) |

### Why Amortized O(1) for Append?

Most appends just write to the next slot — O(1). Occasionally you trigger a resize (copy all n elements). If you double the capacity each time, the total cost of n appends is roughly 2n, so each append "pays" O(1) on average.

### Cache Friendliness

Arrays are the most cache-friendly structure. Modern CPUs fetch memory in cache lines (typically 64 bytes). Iterating an array hits the L1 cache nearly every time, while chasing pointers in a linked list causes cache misses. This constant factor matters enormously in practice — an array-based solution can be 10-100x faster than a pointer-based one for the same Big-O.

```python
# Dynamic array growth in Python
import sys

lst = []
prev_size = 0
for i in range(64):
    current_size = sys.getsizeof(lst)
    if current_size != prev_size:
        print(f"len={len(lst):3d}  capacity_bytes={current_size}")
        prev_size = current_size
    lst.append(i)
```

---

## Linked Lists

### Core Concept

A **linked list** stores elements in nodes scattered across memory, each pointing to the next (and possibly the previous, in a doubly-linked list). Insertions and deletions at a known position are O(1) — just rewire pointers — but finding that position requires O(n) traversal.

### When to Use (and When Not To)

Linked lists are rarely the best choice for general-purpose work. Their main advantages are:

- **O(1) insert/delete at head** (singly-linked) or at any known node (doubly-linked)
- **No resize cost** — each node is independently allocated
- **Building block** for other structures — stacks, queues, hash table chaining, LRU caches

In practice, prefer arrays/dynamic arrays unless you specifically need cheap insertion at arbitrary positions with a reference to the node.

```python
class Node:
    def __init__(self, val, next=None):
        self.val = val
        self.next = next

# Insert at head: O(1)
head = Node(3, Node(2, Node(1)))
head = Node(4, head)  # 4 -> 3 -> 2 -> 1
```

---

## Stacks, Queues, and Deques

These are **abstract data types** — they define an interface, not an implementation.

### Stack (LIFO — Last In, First Out)

- `push(x)`: add to top
- `pop()`: remove from top
- Use cases: function call stack, undo systems, DFS, expression parsing, balanced parentheses

### Queue (FIFO — First In, First Out)

- `enqueue(x)`: add to back
- `dequeue()`: remove from front
- Use cases: BFS, task scheduling, message queues, buffering

### Deque (Double-Ended Queue)

- Insert/remove at both ends in O(1)
- Python's `collections.deque` is implemented as a doubly-linked list of fixed-size blocks
- Use cases: sliding window problems, work-stealing schedulers

```python
from collections import deque

# Stack behavior
stack = []
stack.append(1)      # push
stack.append(2)
stack.pop()          # 2

# Queue behavior
queue = deque()
queue.append(1)      # enqueue
queue.append(2)
queue.popleft()      # 1 (dequeue)

# Deque: both ends
d = deque()
d.appendleft(1)      # front
d.append(2)          # back
d.popleft()          # 1
d.pop()              # 2
```

---

## Hash Tables

### Core Concept

A **hash table** maps keys to values by computing a hash function on the key, which produces an index into an array of "buckets." Average-case O(1) for insert, lookup, and delete.

### Collision Resolution

Two keys can hash to the same bucket. The two main strategies:

**Chaining**: Each bucket holds a linked list (or small array) of entries. On collision, append to the list. Simple, robust, but pointer-chasing hurts cache performance.

**Open addressing**: All entries live in the array itself. On collision, probe for the next open slot (linear probing, quadratic probing, or double hashing). Better cache performance but more sensitive to load factor.

### Load Factor and Rehashing

The **load factor** α = n / capacity (number of entries / number of buckets). As α grows:
- Chaining: average chain length = α, so lookups degrade linearly
- Open addressing: clustering worsens rapidly; performance degrades around α > 0.7

When α exceeds a threshold (commonly 0.75), the table **rehashes**: allocate a larger array (typically 2x) and re-insert all entries. This is O(n) but amortized over many insertions.

### Practical Considerations

- **Hash function quality matters**: a bad hash function creates clusters and destroys O(1) performance
- Python `dict` uses open addressing with a sophisticated probing sequence
- Java `HashMap` uses chaining with linked lists that convert to trees (red-black) when chains get long (> 8)

```python
# Simple hash table implementation sketch
class SimpleHashTable:
    def __init__(self, capacity=8):
        self.capacity = capacity
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]

    def _hash(self, key):
        return hash(key) % self.capacity

    def put(self, key, value):
        idx = self._hash(key)
        for i, (k, v) in enumerate(self.buckets[idx]):
            if k == key:
                self.buckets[idx][i] = (key, value)
                return
        self.buckets[idx].append((key, value))
        self.size += 1
        if self.size / self.capacity > 0.75:
            self._rehash()

    def _rehash(self):
        old = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0
        for bucket in old:
            for key, value in bucket:
                self.put(key, value)
```

---

## Trees

### Binary Search Tree (BST)

A BST maintains the invariant: for every node, all values in the left subtree are smaller, all in the right are larger. This gives O(log n) search, insert, and delete — **if the tree is balanced**. An unbalanced BST degrades to a linked list (O(n)).

### Self-Balancing Trees: AVL and Red-Black

**AVL trees** enforce strict balance: the heights of left and right subtrees differ by at most 1. They perform rotations on insert/delete to maintain this. Lookups are slightly faster than red-black trees, but insertions/deletions are more expensive due to stricter balancing.

**Red-Black trees** use a coloring scheme with looser balance guarantees (height at most 2 log n). Fewer rotations on insert/delete, making them preferred for general-purpose use. Java `TreeMap`, C++ `std::map`, and Linux's CFS scheduler all use red-black trees.

### B-Trees and B+Trees (Databases)

B-Trees generalize BSTs: each node holds multiple keys and has multiple children. A node with m keys has m+1 children. This is designed for **disk-based storage**: each node fits in a disk page (e.g., 4-16 KB), minimizing I/O operations.

**B+Trees** are the variant used by virtually all relational databases (PostgreSQL, MySQL/InnoDB):
- All values live in leaf nodes
- Leaf nodes are linked together for efficient range scans
- Internal nodes only store keys for navigation

Why B+Trees dominate databases: a B+Tree with branching factor 100 and height 4 can index ~100 million records with at most 4 disk reads.

### Tries (Prefix Trees)

A trie stores strings character-by-character along edges. Each path from root to a node represents a prefix. Used for:
- Autocomplete
- Spell checkers
- IP routing tables (longest prefix match)
- String matching in general

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
        node.is_end = True

    def search(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return node.is_end

    def starts_with(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return True
```

---

## Heaps and Priority Queues

### Core Concept

A **binary heap** is a complete binary tree stored as an array where every parent is ≤ its children (min-heap) or ≥ its children (max-heap). The minimum (or maximum) is always at the root.

| Operation | Complexity |
|---|---|
| Find min/max | O(1) |
| Insert | O(log n) |
| Extract min/max | O(log n) |
| Build heap from array | O(n) |

### Array Representation

For a node at index `i`: left child is `2i + 1`, right child is `2i + 2`, parent is `(i - 1) // 2`. No pointers needed — the implicit structure saves memory and is cache-friendly.

### Use Cases

- **Priority queues**: process tasks by priority, not arrival order
- **Heap sort**: O(n log n) in-place sorting
- **Top-k problems**: maintain a heap of size k
- **Dijkstra's algorithm**: extract the closest unvisited vertex
- **Median maintenance**: one max-heap for the lower half, one min-heap for the upper half

```python
import heapq

# Python heapq is a min-heap
nums = [5, 3, 8, 1, 2]
heapq.heapify(nums)          # O(n) — turns list into heap in-place
heapq.heappush(nums, 0)      # insert
smallest = heapq.heappop(nums)  # extract min -> 0

# Top-k largest elements
top3 = heapq.nlargest(3, [5, 3, 8, 1, 2])  # [8, 5, 3]
```

---

## Graph Representations

### Adjacency List

Each vertex stores a list of its neighbors. Space: O(V + E). Best for sparse graphs (most real-world graphs).

### Adjacency Matrix

A V×V matrix where `matrix[i][j] = 1` (or edge weight) if there's an edge. Space: O(V²). Best for dense graphs or when you need O(1) edge existence checks.

### Comparison

| | Adjacency List | Adjacency Matrix |
|---|---|---|
| Space | O(V + E) | O(V²) |
| Check edge (u,v) | O(degree(u)) | O(1) |
| Iterate neighbors | O(degree(u)) | O(V) |
| Add edge | O(1) | O(1) |
| Best for | Sparse graphs, traversals | Dense graphs, edge queries |

```python
# Adjacency list (most common)
from collections import defaultdict

graph = defaultdict(list)
graph[0].append(1)
graph[0].append(2)
graph[1].append(2)

# Adjacency matrix
V = 4
matrix = [[0] * V for _ in range(V)]
matrix[0][1] = 1  # edge 0 -> 1
matrix[0][2] = 1  # edge 0 -> 2

# Weighted adjacency list
weighted_graph = defaultdict(list)
weighted_graph[0].append((1, 5))   # edge 0->1, weight 5
weighted_graph[0].append((2, 3))   # edge 0->2, weight 3
```

---

## Probabilistic and Specialized Structures

### Bloom Filters

A **Bloom filter** answers "is this element in the set?" with:
- **Definite no** — if it says no, the element is definitely not in the set
- **Probable yes** — if it says yes, the element is *probably* in the set (false positives possible)

How it works: k hash functions each set a bit in a bit array on insert. To query, check if all k bits are set. False positive rate depends on array size, number of hash functions, and number of elements.

Use cases: spell checkers, cache lookups (avoid expensive checks for absent keys), network routers, databases (check if an SSTable might contain a key before reading it from disk).

### Skip Lists

A **skip list** is a layered linked list with express lanes. Each element appears in the bottom layer; with probability p (typically 0.5), it also appears in the layer above. This creates a hierarchy that gives O(log n) search, insert, and delete — like a balanced BST but simpler to implement.

Used in Redis sorted sets and LevelDB's memtable.

---

## Storage-Oriented Structures

### LSM Trees (Log-Structured Merge Trees)

LSM trees optimize for **write-heavy workloads** by buffering writes in memory (a memtable, often a skip list or red-black tree), then flushing sorted runs to disk. Background compaction merges sorted runs.

**Write path**: write to in-memory buffer → when buffer is full, flush as sorted file (SSTable) → background merge/compaction

**Read path**: check memtable → check each level of SSTables (Bloom filters speed this up)

Trade-off: excellent write throughput at the cost of read amplification (may need to check multiple levels). Used by LevelDB, RocksDB, Cassandra, HBase.

### Roaring Bitmaps

Standard bitmaps waste space on sparse data. **Roaring bitmaps** partition the value space into chunks of 2^16 integers and store each chunk using the best representation (array, bitmap, or run-length encoding based on density).

Used in analytics engines (Apache Druid, ClickHouse), search engines (Lucene/Elasticsearch), and columnar databases for fast set operations (AND/OR/XOR) on large integer sets.

---

## Persistent and Immutable Data Structures

In functional programming, you want data structures that preserve previous versions after modification. A **persistent** structure shares most of its content between versions via **structural sharing**.

Example: a persistent balanced tree. When you "modify" a node, you create a new node and new ancestors up to the root, but reuse all unaffected subtrees. Cost: O(log n) time and space per update, but you keep full version history.

Practical uses:
- **Git**: the object store is essentially a persistent Merkle tree
- **Clojure/Scala collections**: persistent vectors, maps, and sets using hash array mapped tries (HAMTs)
- **React/Redux**: immutable state enables efficient change detection
- **Database MVCC**: multiple readers see consistent snapshots without locking

---

## Concurrency-Aware Data Structures

### Lock-Free Queues

A **lock-free queue** uses atomic compare-and-swap (CAS) operations instead of mutexes. At least one thread always makes progress, even if other threads are delayed. The Michael-Scott queue is the classic algorithm.

Key ideas:
- Nodes are linked with atomic pointers
- Enqueue: CAS the tail's `next` pointer, then CAS the tail itself
- Dequeue: CAS the head pointer forward
- ABA problem: solved with tagged pointers or hazard pointers

### Concurrent Hash Maps

A **concurrent hash map** supports multiple readers and writers without a global lock. Common strategies:

- **Striped locking**: divide buckets into segments, each with its own lock (Java `ConcurrentHashMap` pre-Java 8)
- **Lock-free reads with CAS writes**: Java 8+ `ConcurrentHashMap` uses CAS for updates and volatile reads
- **Read-copy-update (RCU)**: readers proceed without synchronization; writers create a modified copy and atomically swap (used in Linux kernel)

These are essential in high-throughput systems — a single global lock on a hash map becomes a bottleneck under contention.

---

## Exercises

1. **Array vs. Linked List**: You need to implement an LRU cache with O(1) get and put. Which data structures would you combine and why? Sketch the design.

2. **Hash Table Analysis**: A hash table with chaining has 16 buckets and 12 entries. What is the load factor? If one bucket has 5 entries while most have 0-1, what does this suggest about the hash function?

3. **Tree Selection**: For each scenario, choose the most appropriate tree structure and justify:
   - A database index supporting range queries on timestamps
   - An autocomplete system for a search engine
   - An in-memory ordered map in a general-purpose application

4. **Heap Application**: Given a stream of integers, maintain the running median. Describe your approach using heaps. What are the time complexities for adding a number and querying the median?

5. **Bloom Filter Trade-offs**: You want to check if a URL has been visited before. You have 10 million URLs and can afford a 1% false positive rate. Why is a Bloom filter preferable to a hash set here? What happens if you need to delete URLs?

6. **Graph Representation**: You're modeling a social network with 1 billion users where the average user has 200 friends. Would you use an adjacency list or matrix? Calculate the approximate memory usage for each representation.

7. **Concurrency Design**: A web server processes 100k requests/second and needs a shared request counter. Why would a lock-free approach outperform a mutex-based one? What hardware feature makes lock-free structures possible?

---

## Recommended Resources

- **Books**:
  - *Introduction to Algorithms* (CLRS) — chapters 10-22 cover all fundamental structures
  - *Designing Data-Intensive Applications* (Kleppmann) — chapters 3 on LSM trees, B-trees, and storage engines
  - *The Art of Multiprocessor Programming* (Herlihy & Shavit) — lock-free structures

- **Interactive**:
  - [VisuAlgo](https://visualgo.net/) — animated visualizations of data structures and algorithms
  - [Data Structure Visualizations](https://www.cs.usfca.edu/~galles/visualization/) — University of San Francisco interactive tool

- **Practice**:
  - LeetCode problems tagged by data structure (start with Easy, progress to Medium)
  - Implement a hash table, a heap, and a trie from scratch in your language of choice
  - Read the source code of your language's standard library collections

- **Deep Dives**:
  - [The Log-Structured Merge-Tree](https://www.cs.umb.edu/~pon} ] /lsm.pdf) — original O'Neil et al. paper
  - [Roaring Bitmaps paper](https://arxiv.org/abs/1603.06549) — design and benchmarks
  - [Java ConcurrentHashMap internals](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html) — Javadoc with implementation notes
