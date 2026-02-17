# Discrete Math — The Core Language of CS

Discrete math is the foundation that the rest of computer science is built on. Every time you reason about whether a program terminates, analyze an algorithm's complexity, model a network as a graph, or verify a security protocol — you're doing discrete math. You don't need to be a mathematician, but you need fluency in these concepts the same way a writer needs fluency in grammar: not to show off, but to think clearly.

---

## 1. Logic

### Why it matters

Logic is how we make precise statements about programs. "If the user is authenticated **and** has the admin role, grant access" is propositional logic. Database query optimizers rewrite your SQL using logical equivalences. Type checkers prove properties about your code using logical rules.

### Core concepts

- **Propositions** — statements that are either true or false. "The server is running" is a proposition. "Close the door" is not.
- **Logical connectives** — AND (`∧`), OR (`∨`), NOT (`¬`), implication (`→`), biconditional (`↔`)
- **Implication (`P → Q`)** — "if P then Q." Only false when P is true and Q is false. A crucial subtlety: `false → anything` is true (vacuous truth). This is why an empty `for` loop over zero items "correctly" satisfies any postcondition.
- **De Morgan's Laws** — `¬(P ∧ Q) ≡ ¬P ∨ ¬Q` and `¬(P ∨ Q) ≡ ¬P ∧ ¬Q`. You use these every time you refactor a conditional:

```javascript
// Before: !(isAdmin && isActive)
// After (De Morgan): !isAdmin || !isActive
if (!user.isAdmin || !user.isActive) {
  return "Access denied";
}
```

- **Predicates** — propositions with variables: `isEven(x)` is true when `x` is even. Predicates turn logic into something you can apply to collections of objects.
- **Quantifiers** — `∀x` (for all x) and `∃x` (there exists an x). "All users have unique emails" is `∀u1, ∀u2: u1 ≠ u2 → email(u1) ≠ email(u2)`. The negation of "all" is "there exists one that doesn't": `¬(∀x P(x)) ≡ ∃x ¬P(x)`.

### Practical application

When you write a database constraint `UNIQUE(email)`, you're encoding `∀u1, ∀u2`. When your test asserts "no item in the list is negative," you're asserting `∀x ∈ list: x ≥ 0`. Thinking in quantifiers helps you spot edge cases — the negation tells you exactly what a counterexample looks like.

---

## 2. Proofs

### Why it matters

You don't write formal proofs at work, but you use proof techniques constantly. Arguing that your algorithm is correct, that an invariant holds across loop iterations, or that a recursive function terminates — these are all informal proofs. Knowing the techniques makes your reasoning sharper and your code reviews more effective.

### Core techniques

- **Direct proof** — assume P, derive Q step by step. "If `n` is even, then `n²` is even." Assume `n = 2k`, then `n² = 4k² = 2(2k²)`, which is even.
- **Proof by contradiction** — assume the opposite, derive something absurd. Classic: "√2 is irrational." Assume it's rational (`p/q` in lowest terms), derive that both `p` and `q` must be even — contradicting "lowest terms."
- **Proof by contrapositive** — instead of proving `P → Q`, prove `¬Q → ¬P` (logically equivalent). Often easier. "If `n²` is odd, then `n` is odd" is easier to prove as: "if `n` is even, then `n²` is even."
- **Mathematical induction** — prove a base case, then prove "if it holds for `k`, it holds for `k+1`." This is exactly how you reason about recursive functions:

```python
def factorial(n):
    if n == 0:        # Base case: 0! = 1 ✓
        return 1
    return n * factorial(n - 1)  # Inductive step: n! = n × (n-1)!
```

- **Loop invariants** — a condition that is true before the loop, maintained by each iteration, and gives you the desired result when the loop ends. This is how you convince yourself (or a colleague) that a loop is correct.

```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    # Invariant: if target exists, it's in arr[lo..hi]
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1    # Invariant maintained: target can't be in arr[..mid]
        else:
            hi = mid - 1    # Invariant maintained: target can't be in arr[mid..]
    return -1  # lo > hi, so the range is empty — target doesn't exist
```

---

## 3. Sets

### Why it matters

Sets are the building blocks for modeling data. Database tables are sets of rows. Types are sets of values. Access control lists are sets of permissions. Understanding set operations makes you better at SQL, API design, and data modeling.

### Core concepts

- **Basic operations** — union (`A ∪ B`), intersection (`A ∩ B`), difference (`A \ B`), complement (`Ā`)
- **Relations** — a set of ordered pairs. A database table with columns `(user_id, role)` defines a relation between users and roles.
- **Properties of relations**:
  - *Reflexive*: every element relates to itself (`a R a`)
  - *Symmetric*: if `a R b` then `b R a` (like "is friends with")
  - *Transitive*: if `a R b` and `b R c` then `a R c` (like "is ancestor of")
  - *Antisymmetric*: if `a R b` and `b R a` then `a = b` (like "≤")
- **Equivalence relations** — reflexive + symmetric + transitive. They partition a set into equivalence classes. Example: "same remainder when divided by 3" partitions integers into three classes: `{..., -3, 0, 3, 6, ...}`, `{..., -2, 1, 4, 7, ...}`, `{..., -1, 2, 5, 8, ...}`.
- **Functions** — a relation where each input maps to exactly one output. Injective (one-to-one), surjective (onto), bijective (both). Hash functions are not injective (collisions exist). A good encryption function is bijective (reversible).

### Practical application

SQL is set theory with syntax: `UNION`, `INTERSECT`, `EXCEPT` map directly. Understanding functions as relations helps you reason about database normalization — a table is in good form when its functional dependencies are clean.

---

## 4. Combinatorics

### Why it matters

Combinatorics tells you "how many ways" something can happen. This is essential for analyzing algorithm complexity, estimating search spaces, calculating probabilities in load balancing, and understanding why brute force is sometimes hopeless.

### Core concepts

- **Permutations** — ordered arrangements. `n!` ways to arrange `n` items. 10 items = 3,628,800 arrangements. 20 items = ~2.4 × 10¹⁸ (forget brute force).
- **Combinations** — unordered selections. `C(n, k) = n! / (k!(n-k)!)`. "Choose 3 servers from 10" = `C(10,3)` = 120 ways.
- **The Pigeonhole Principle** — if you put `n+1` pigeons into `n` holes, at least one hole has ≥ 2 pigeons. Sounds trivial, but it's powerful:
  - If you hash 366 items into 365 buckets, at least one collision is **guaranteed**
  - If a function maps from a larger domain to a smaller codomain, it can't be injective
  - In any group of 13 people, at least 2 share a birth month

### Practical application

Password strength: a 4-digit PIN has `10⁴ = 10,000` combinations. An 8-character lowercase password has `26⁸ ≈ 209 billion`. Adding uppercase + digits: `62⁸ ≈ 218 trillion`. Combinatorics quantifies why longer, varied passwords matter.

---

## 5. Graph Theory

### Why it matters

Graphs are everywhere in computing: networks, dependency trees, social connections, state machines, database schemas, file system directories. The moment you have entities with relationships, you have a graph. Knowing graph vocabulary lets you recognize problems that already have known solutions.

### Core concepts

- **Graph** — vertices (nodes) + edges (connections). Directed or undirected. Weighted or unweighted.
- **Trees** — connected acyclic graphs. A tree with `n` nodes has exactly `n-1` edges. File systems, DOM, ASTs, and B-Trees are all trees.
- **DAGs (Directed Acyclic Graphs)** — directed graphs with no cycles. Build systems (Make, Webpack), task schedulers, and `git` commit histories are DAGs. You can always topologically sort a DAG.
- **Cycles** — a path that starts and ends at the same vertex. Cycles in dependency graphs mean deadlocks or circular imports. Detecting cycles is a fundamental graph operation (DFS with back-edge detection).
- **Connectivity** — a graph is connected if there's a path between every pair of nodes. In directed graphs, **strongly connected components** (SCCs) are maximal subsets where every node can reach every other.
- **Bipartite graphs** — vertices split into two groups where edges only go between groups. Matching problems (assign tasks to workers, match users to servers) live on bipartite graphs.
- **Matching** — a set of edges with no shared vertices. Maximum matching in bipartite graphs has efficient algorithms (Hopcroft-Karp). Practical use: optimal assignment problems.

### Practical application

When you model microservices as nodes and API calls as edges, you can check for circular dependencies (cycle detection), find which services are tightly coupled (SCCs), and determine build/deploy order (topological sort).

---

## 6. Number Theory

### Why it matters

Number theory powers cryptography, hashing, and error detection. Every HTTPS connection you make relies on modular arithmetic and prime numbers. Understanding the basics helps you reason about hash functions, random number generation, and why RSA works.

### Core concepts

- **Modular arithmetic** — clock arithmetic. `17 mod 12 = 5` (5 o'clock). Key properties: `(a + b) mod n = ((a mod n) + (b mod n)) mod n`. Same for multiplication. This is why hash tables use `hash(key) % table_size`.
- **GCD (Greatest Common Divisor)** — the largest number dividing both `a` and `b`. Euclid's algorithm is elegant and fast:

```python
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a
# gcd(48, 18) → gcd(18, 12) → gcd(12, 6) → gcd(6, 0) → 6
```

- **Primes** — numbers divisible only by 1 and themselves. The **Fundamental Theorem of Arithmetic** says every integer > 1 has a unique prime factorization. RSA encryption relies on the fact that multiplying two large primes is easy, but factoring their product is computationally infeasible.
- **Modular inverse** — `a⁻¹ mod n` exists iff `gcd(a, n) = 1`. Used in RSA decryption and modular division.
- **Fermat's Little Theorem** — if `p` is prime and `gcd(a, p) = 1`, then `a^(p-1) ≡ 1 (mod p)`. Basis for primality testing and key generation.

### Practical application

Hash maps use modular arithmetic for bucket assignment. Cryptographic protocols (TLS, SSH) use prime-based key exchange. Checksums and error-correcting codes use modular arithmetic for integrity verification.

---

## Exercises

1. **Logic**: Write the negation of "All servers in the cluster are healthy and reachable." Apply De Morgan's and quantifier negation rules.
2. **Proofs**: Prove by induction that `1 + 2 + ... + n = n(n+1)/2`.
3. **Proofs**: Identify the loop invariant in a simple `find_max(array)` function and argue why it's correct.
4. **Sets**: Given sets `A = {1,2,3,4,5}` and `B = {3,4,5,6,7}`, compute `A ∪ B`, `A ∩ B`, `A \ B`, and `B \ A`. Translate these into SQL using `UNION`, `INTERSECT`, and `EXCEPT`.
5. **Combinatorics**: A system generates 8-character API keys from `[a-zA-Z0-9]`. How many possible keys exist? If the system has 10 million active keys, what is the probability of a collision when generating one new key?
6. **Graphs**: Draw the dependency graph for these build targets: `A → B, A → C, B → D, C → D, D → E`. Is it a DAG? Give a valid topological ordering.
7. **Number theory**: Compute `gcd(252, 198)` using Euclid's algorithm (show each step). Then verify that `252 × 198 = gcd × lcm`.

---

## Recommended Resources

- **Book**: *Discrete Mathematics and Its Applications* by Kenneth Rosen — the standard textbook; comprehensive reference
- **Book**: *Mathematics for Computer Science* by Lehman, Leighton, Meyer (MIT OpenCourseWare, free PDF) — excellent and free
- **Course**: MIT 6.042J Mathematics for Computer Science (OCW) — video lectures matching the free textbook
- **Interactive**: *Brilliant.org* Discrete Math courses — great for building intuition through problems
- **Reference**: *Concrete Mathematics* by Graham, Knuth, Patashnik — deeper dive, especially combinatorics and number theory
- **Practice**: Project Euler (projecteuler.net) — number theory and combinatorics problems that require code

---

*Next lesson: [Computer Architecture](./02_computer_architecture.md)*
