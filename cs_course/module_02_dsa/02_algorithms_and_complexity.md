# Algorithms and Complexity

An algorithm is a finite sequence of well-defined instructions that transforms input into output. This lesson covers the tools for analyzing how efficient algorithms are (complexity analysis), the major design paradigms for constructing algorithms, the most important problems and their solutions, and the theoretical limits of computation. The goal is not to memorize solutions but to develop the ability to recognize patterns, choose strategies, and reason about trade-offs.

---

## Big-O and Complexity Analysis

### What Big-O Actually Means

Big-O notation describes how an algorithm's resource usage (time or space) grows as input size n grows, ignoring constant factors and lower-order terms. Formally, f(n) = O(g(n)) means there exist constants c and n₀ such that f(n) ≤ c·g(n) for all n ≥ n₀.

It answers: **"As the input gets very large, how does the cost scale?"**

### Common Complexity Classes

| Class | Name | Example |
|---|---|---|
| O(1) | Constant | Array access, hash table lookup |
| O(log n) | Logarithmic | Binary search, balanced BST lookup |
| O(n) | Linear | Linear scan, single pass over array |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(2ⁿ) | Exponential | All subsets, naive recursive Fibonacci |
| O(n!) | Factorial | All permutations, brute-force TSP |

Intuition for scale (n = 1,000,000):
- O(n log n) ≈ 20 million operations — runs in milliseconds
- O(n²) = 1 trillion operations — runs in hours
- O(2ⁿ) — heat death of the universe wouldn't be enough

### Time vs. Space Trade-offs

Many problems offer a choice: use more memory to save time, or vice versa.

- **Memoization**: store computed results to avoid recomputation (trade space for time)
- **Hash tables**: O(n) extra space gives O(1) lookups instead of O(n)
- **Bit manipulation**: compact representation saves space at the cost of complexity
- **Streaming algorithms**: process data in one pass with O(1) space, sacrificing exactness

### Amortized Analysis

Some operations are expensive occasionally but cheap most of the time. **Amortized analysis** averages the cost over a sequence of operations.

Classic example: dynamic array append. Most appends are O(1), but occasionally you resize (O(n)). Over n appends, total work is O(n), so amortized cost per append is O(1).

Methods for amortized analysis:
- **Aggregate method**: total cost / number of operations
- **Banker's method**: assign "credits" to cheap operations that pay for expensive ones
- **Physicist's method**: define a potential function that tracks stored work

### Constant Factors and Cache Effects

Big-O hides constant factors, but in practice they matter enormously:

- **Cache locality**: iterating an array (sequential memory access) can be 100x faster than traversing a linked list (random memory access), even with the same Big-O
- **Branch prediction**: predictable branches (e.g., sorted data) execute much faster
- **SIMD/vectorization**: operating on contiguous data lets the CPU process multiple elements at once
- **Algorithmic constants**: O(n log n) merge sort with large constants can lose to O(n²) insertion sort for small n (which is why practical sorts like Timsort use insertion sort for small subarrays)

```python
import time

# Same O(n) traversal, vastly different real-world performance
n = 10_000_000

# Sequential access (cache-friendly)
arr = list(range(n))
start = time.time()
total = sum(arr)
print(f"Array sum: {time.time() - start:.3f}s")

# Random access (cache-hostile)
import random
indices = list(range(n))
random.shuffle(indices)
start = time.time()
total = sum(arr[i] for i in indices)
print(f"Random access sum: {time.time() - start:.3f}s")
```

---

## Algorithm Design Patterns

### Divide and Conquer

**Strategy**: Break the problem into smaller subproblems of the same type, solve them recursively, then combine the results.

**When to use**: The problem has natural recursive structure and subproblems don't overlap.

Classic examples:
- **Merge sort**: split array in half, sort each half, merge — O(n log n)
- **Quick sort**: partition around a pivot, sort each side — O(n log n) average
- **Binary search**: eliminate half the search space each step — O(log n)
- **Karatsuba multiplication**: multiply large numbers faster than school method — O(n^1.585)

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

### Greedy Algorithms

**Strategy**: At each step, make the locally optimal choice, hoping it leads to a globally optimal solution.

**When to use**: The problem has the **greedy choice property** (a locally optimal choice is part of some globally optimal solution) and **optimal substructure**.

**Warning**: Greedy doesn't always work. You must prove (or at least argue) correctness.

Classic examples:
- **Activity selection**: pick the activity that finishes earliest — O(n log n)
- **Huffman coding**: build optimal prefix codes by merging lowest-frequency nodes
- **Dijkstra's algorithm**: always expand the closest unvisited vertex (greedy on shortest known distance)
- **Kruskal's/Prim's MST**: add the cheapest edge that doesn't create a cycle

```python
def activity_selection(activities):
    """Select maximum non-overlapping activities.
    activities: list of (start, end) tuples."""
    activities.sort(key=lambda x: x[1])  # sort by end time
    selected = [activities[0]]
    for start, end in activities[1:]:
        if start >= selected[-1][1]:
            selected.append((start, end))
    return selected
```

### Dynamic Programming (DP)

**Strategy**: Break the problem into overlapping subproblems, solve each once, and store results to avoid recomputation.

**When to use**: The problem has **optimal substructure** (optimal solution is built from optimal solutions to subproblems) and **overlapping subproblems** (same subproblems recur many times).

Two approaches:
- **Top-down (memoization)**: recursive with a cache — more intuitive, only computes needed subproblems
- **Bottom-up (tabulation)**: iterative, fill a table from smallest subproblems up — avoids recursion overhead, easier to optimize space

```python
# Fibonacci: the canonical DP example

# Naive recursive: O(2^n) — exponential!
def fib_naive(n):
    if n <= 1: return n
    return fib_naive(n - 1) + fib_naive(n - 2)

# Top-down with memoization: O(n)
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n <= 1: return n
    return fib_memo(n - 1) + fib_memo(n - 2)

# Bottom-up tabulation: O(n) time, O(1) space
def fib_tab(n):
    if n <= 1: return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

Common DP problems: longest common subsequence, edit distance, knapsack (0/1 and unbounded), coin change, matrix chain multiplication, longest increasing subsequence.

### Backtracking

**Strategy**: Build a solution incrementally, abandoning ("backtracking" from) a partial solution as soon as you determine it cannot lead to a valid/optimal complete solution.

**When to use**: Constraint satisfaction, combinatorial search, when you need all solutions (or any valid solution).

```python
def solve_n_queens(n):
    """Find all valid placements of n queens on an n×n board."""
    solutions = []

    def backtrack(row, cols, diag1, diag2, board):
        if row == n:
            solutions.append(["".join(r) for r in board])
            return
        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue
            board[row][col] = "Q"
            backtrack(row + 1, cols | {col}, diag1 | {row - col},
                      diag2 | {row + col}, board)
            board[row][col] = "."

    board = [["." for _ in range(n)] for _ in range(n)]
    backtrack(0, set(), set(), set(), board)
    return solutions
```

---

## Algorithmic Techniques (Problem-Solving Toolkit)

### Two Pointers

Use two indices that move toward each other (or in the same direction) to process a sorted array or linked list in O(n).

```python
def two_sum_sorted(arr, target):
    """Find two numbers in a sorted array that sum to target."""
    left, right = 0, len(arr) - 1
    while left < right:
        s = arr[left] + arr[right]
        if s == target:
            return (left, right)
        elif s < target:
            left += 1
        else:
            right -= 1
    return None
```

### Sliding Window

Maintain a window over a contiguous subarray, expanding/shrinking it to satisfy constraints. Converts O(n²) brute-force to O(n).

```python
def max_sum_subarray(arr, k):
    """Maximum sum of any contiguous subarray of length k."""
    window_sum = sum(arr[:k])
    max_sum = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]
        max_sum = max(max_sum, window_sum)
    return max_sum
```

### Prefix Sums

Precompute cumulative sums so any subarray sum can be answered in O(1).

```python
def build_prefix_sums(arr):
    prefix = [0] * (len(arr) + 1)
    for i in range(len(arr)):
        prefix[i + 1] = prefix[i] + arr[i]
    return prefix

# Sum of arr[l..r] inclusive = prefix[r+1] - prefix[l]
```

### Union-Find (Disjoint Set Union)

Tracks a partition of elements into disjoint sets. With **union by rank** and **path compression**, both `find` and `union` run in nearly O(1) — specifically O(α(n)), where α is the inverse Ackermann function.

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx == ry: return False
        if self.rank[rx] < self.rank[ry]:
            rx, ry = ry, rx
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]:
            self.rank[rx] += 1
        return True
```

Use cases: Kruskal's MST, connected components, cycle detection, network connectivity.

### Randomized Algorithms

Introduce randomness to achieve good expected performance or simplify the algorithm.

- **Randomized quicksort**: random pivot gives O(n log n) expected time regardless of input
- **Reservoir sampling**: select k items uniformly at random from a stream of unknown length
- **Monte Carlo methods**: probabilistic results (e.g., randomized primality testing)
- **Las Vegas algorithms**: always correct, randomized running time (e.g., randomized quicksort)

### Approximation Algorithms

For NP-hard problems, find a solution guaranteed to be within some factor of optimal.

- **Vertex cover**: greedy 2-approximation (always within 2x of optimal)
- **Traveling salesman (metric)**: Christofides' algorithm gives 1.5-approximation
- **Set cover**: greedy gives O(log n)-approximation (and this is nearly the best possible)

---

## Core Problems

### Sorting and Searching

| Algorithm | Time (avg) | Time (worst) | Space | Stable? | Notes |
|---|---|---|---|---|---|
| Insertion sort | O(n²) | O(n²) | O(1) | Yes | Fast for small/nearly sorted inputs |
| Merge sort | O(n log n) | O(n log n) | O(n) | Yes | Predictable, parallelizable |
| Quick sort | O(n log n) | O(n²) | O(log n) | No | Fastest in practice (cache-friendly) |
| Heap sort | O(n log n) | O(n log n) | O(1) | No | In-place, but poor cache behavior |
| Counting/Radix | O(n + k) | O(n + k) | O(n + k) | Yes | Non-comparison; works for bounded integers |

**Binary search** is the workhorse of searching in sorted data. Know these variations: find exact match, find first/last occurrence, find insertion point, search on answer (binary search on the solution space).

**Selection (kth element)**: Quickselect gives O(n) average (O(n²) worst). The median-of-medians algorithm guarantees O(n) worst-case but has large constants.

### String Matching

- **Naive**: O(nm) — check every position
- **KMP (Knuth-Morris-Pratt)**: O(n + m) — precompute a failure function to avoid re-examining characters
- **Rabin-Karp**: O(n + m) expected — use rolling hash; good for multi-pattern search
- **Boyer-Moore**: sublinear in practice — skip characters using bad-character and good-suffix rules

### Graph Algorithms

**BFS (Breadth-First Search)**: explore level by level using a queue. O(V + E). Finds shortest path in unweighted graphs.

**DFS (Depth-First Search)**: explore as deep as possible using a stack (or recursion). O(V + E). Finds connected components, topological order, cycles.

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order

def dfs(graph, start):
    visited = set()
    order = []
    def explore(node):
        visited.add(node)
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                explore(neighbor)
    explore(start)
    return order
```

**Shortest Paths**:
- **Dijkstra**: single-source, non-negative weights — O((V + E) log V) with a min-heap
- **Bellman-Ford**: single-source, allows negative weights — O(VE), detects negative cycles
- **Floyd-Warshall**: all-pairs shortest paths — O(V³)

**Minimum Spanning Tree (MST)**:
- **Kruskal's**: sort edges, add cheapest non-cycle-creating edge — O(E log E), uses Union-Find
- **Prim's**: grow tree from a vertex, always add cheapest adjacent edge — O((V + E) log V) with a heap

**Topological Sort**: linear ordering of vertices in a DAG such that every edge goes from earlier to later. Two approaches: Kahn's algorithm (BFS-based, using in-degree) or DFS post-order (reverse finish order).

```python
def topological_sort_kahn(graph, num_vertices):
    """Kahn's algorithm using in-degree tracking."""
    in_degree = [0] * num_vertices
    for u in graph:
        for v in graph[u]:
            in_degree[v] += 1

    queue = deque(v for v in range(num_vertices) if in_degree[v] == 0)
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph.get(node, []):
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    if len(order) != num_vertices:
        raise ValueError("Graph has a cycle")
    return order
```

**Strongly Connected Components (SCC)**: maximal subsets of vertices where every vertex is reachable from every other. Tarjan's algorithm or Kosaraju's algorithm, both O(V + E).

**Network Flows and Matching**: Ford-Fulkerson method finds maximum flow in a network. Maximum bipartite matching reduces to max-flow. These appear in resource allocation, scheduling, and network routing.

---

## Complexity Theory

### P vs NP

- **P**: problems solvable in polynomial time (e.g., sorting, shortest path, matching)
- **NP**: problems whose solutions can be *verified* in polynomial time (e.g., given a proposed Hamiltonian cycle, check if it's valid in O(n))
- **The question**: Does P = NP? Can every problem whose solution is quickly verifiable also be quickly solvable? Most computer scientists believe P ≠ NP, but this is unproven — it's the most important open problem in CS.

### NP-Complete and NP-Hard

- **NP-hard**: at least as hard as the hardest problems in NP. Solving one in polynomial time would mean P = NP.
- **NP-complete**: NP-hard AND in NP (verifiable in polynomial time). These are the "hardest problems in NP."

Key NP-complete problems: SAT (Boolean satisfiability), 3-SAT, traveling salesman (decision version), graph coloring, subset sum, knapsack (decision version), Hamiltonian cycle, vertex cover, clique.

### Reductions

A **reduction** transforms one problem into another. If problem A reduces to problem B in polynomial time, then:
- If B is easy, A is easy (use B as a subroutine)
- If A is hard, B is hard (B is at least as hard as A)

This is the tool for proving NP-completeness: show your problem is in NP, then reduce a known NP-complete problem to it.

### Decidability and the Halting Problem

Some problems are not just hard — they're **undecidable**: no algorithm can solve them for all inputs, regardless of time allowed.

The **halting problem** is the classic example: "Given a program P and input I, does P eventually halt?" Turing proved (1936) that no general algorithm can decide this. The proof is a beautiful diagonalization argument:

1. Assume a halting decider H(P, I) exists
2. Construct a program D that calls H on itself: if H says "halts," D loops forever; if H says "loops," D halts
3. Running D on itself creates a contradiction — H cannot exist

Practical implication: perfect automated bug detection, program verification, and optimization are fundamentally impossible in general. We must rely on approximations, heuristics, and restricted problem domains.

---

## Exercises

1. **Complexity Analysis**: Determine the time complexity of each code snippet:

```python
# (a)
for i in range(n):
    for j in range(i, n):
        print(i, j)

# (b)
i = n
while i > 0:
    i //= 2

# (c)
def mystery(n):
    if n <= 1: return 1
    return mystery(n // 2) + mystery(n // 2)
```

2. **Greedy vs DP**: The coin change problem asks for the minimum number of coins to make a given amount. Why does greedy work for US coin denominations (1, 5, 10, 25) but fail for denominations like (1, 3, 4) with target 6? Solve the general case with DP.

3. **Sliding Window**: Given an array of integers and a target sum S, find the smallest contiguous subarray with sum ≥ S. Implement an O(n) solution.

4. **Graph Problem**: Given a list of course prerequisites (e.g., "to take course B, you must first take course A"), determine if it's possible to finish all courses. If so, find a valid order. Identify which graph algorithm applies and implement it.

5. **Shortest Path Comparison**: A GPS navigation system needs shortest paths. Compare Dijkstra's and Bellman-Ford for this use case. Which would you choose and why? Are there scenarios where Bellman-Ford would be necessary even in navigation?

6. **NP-Completeness**: You're asked to determine if a social network can be divided into exactly 3 groups such that no two people in the same group are friends. Is this problem in P, NP, or NP-complete? Justify your answer by identifying the equivalent graph theory problem.

7. **Design Challenge**: Design an algorithm that, given a dictionary of words, finds all anagram groups (words that are rearrangements of the same letters). What data structure would you use? What is the time complexity?

---

## Recommended Resources

- **Books**:
  - *Introduction to Algorithms* (CLRS) — the comprehensive reference; chapters 1-35
  - *Algorithm Design Manual* (Skiena) — more practical, with a "war stories" perspective
  - *Algorithms* (Sedgewick & Wayne) — excellent visualizations and Java implementations
  - *Computational Complexity: A Modern Approach* (Arora & Barak) — for deeper complexity theory

- **Courses**:
  - MIT 6.006 Introduction to Algorithms (OCW) — foundational algorithms course
  - MIT 6.046 Design and Analysis of Algorithms (OCW) — advanced techniques
  - Stanford CS161 (Tim Roughgarden) — algorithm design and analysis

- **Practice**:
  - [LeetCode](https://leetcode.com/) — curated problem sets by topic and difficulty
  - [Codeforces](https://codeforces.com/) — competitive programming with editorial explanations
  - [NeetCode 150](https://neetcode.io/) — structured roadmap of core problems
  - [Visualgo](https://visualgo.net/) — step-by-step algorithm visualizations

- **Deep Dives**:
  - [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — quick reference for common operations
  - [Turing's 1936 Paper](https://www.cs.virginia.edu/~robins/Turing_Paper_1936.pdf) — "On Computable Numbers" (the halting problem paper, surprisingly readable)
