# Programming Language Concepts

Programming languages are tools for expressing computation, and they make wildly different tradeoffs. Understanding the concepts that cut across languages — types, memory models, polymorphism, error handling, concurrency — lets you learn new languages faster, choose the right tool for a job, and avoid entire categories of bugs. This lesson covers the ideas, not the syntax.

---

## 1. Compilation vs Interpretation, JIT, Bytecode, VMs

How your code goes from text to execution affects performance, debugging, and deployment. The boundary between "compiled" and "interpreted" is blurrier than most people think.

- **AOT compilation** — source → machine code before execution. C, C++, Rust, Go. Fast execution, slow build, platform-specific.
- **Interpretation** — source read and executed line by line. Classic Python, shell scripts. Fast startup, slow execution.
- **Bytecode + VM** — source → intermediate bytecode → VM executes it. Java (JVM), C# (CLR), Python (CPython). Portable bytecode, VM handles platform differences.
- **JIT (Just-In-Time)** — bytecode compiled to machine code *at runtime*, optimizing hot paths. V8 (JavaScript), HotSpot (Java), PyPy. JIT can sometimes beat AOT because it optimizes based on actual runtime behavior (e.g., inlining a virtual method that's only ever called on one concrete type).

**In practice**: TypeScript compiles to JS (types erased at runtime), V8 JIT-compiles hot functions. CPython interprets bytecode (NumPy is fast because hot loops are in C). JVM JIT achieves near-C performance for long-running servers.

---

## 2. Static vs Dynamic Typing; Soundness; Inference

### Why it matters

Type systems are your first line of defense against bugs. They differ in *when* errors are caught (compile time vs runtime) and *what guarantees* they provide.

### Static vs dynamic

- **Static typing** — types checked at compile time. Java, TypeScript, Rust, Go. Catches errors before execution. Enables better IDE support (autocomplete, refactoring).
- **Dynamic typing** — types checked at runtime. Python, JavaScript, Ruby. More flexible, faster prototyping, but errors show up in production.

### Soundness

A type system is **sound** if it guarantees: "if it compiles, no type errors at runtime." Java and TypeScript are **not** fully sound:

```typescript
// TypeScript unsoundness example: any bypasses all checks
const x: any = "hello";
const y: number = x;  // Compiles fine, y is actually a string at runtime
```

Rust's type system is sound (excluding `unsafe` blocks). Haskell's is sound. TypeScript intentionally trades some soundness for practicality.

### Type inference

Modern languages let you skip type annotations where the compiler can figure them out:

```typescript
// TypeScript infers the type from the value
const name = "Alice";           // inferred: string
const nums = [1, 2, 3];        // inferred: number[]
const double = (x: number) => x * 2;  // return type inferred: number
```

Rust, Kotlin, and Haskell have even stronger inference. More inference = less annotation boilerplate while keeping static safety.

---

## 3. Memory: Stack vs Heap; Pointers/References; Aliasing

Every variable lives somewhere in memory. Where it lives affects performance, lifetime, and what bugs are possible.

- **Stack** — LIFO, automatic allocation/deallocation. Local variables live here. Very fast (just move a pointer). Fixed size (1-8 MB per thread), data must have a known size at compile time.
- **Heap** — dynamic allocation (`new`, `malloc`). Lives until freed or garbage collected. Slower, can fragment, but no size limit.

**Pointers, references, aliasing**:

- **Pointer** — stores a memory address. Can be null, can dangle (point to freed memory). C, C++, Go.
- **Reference** — safer abstraction. Usually can't be null (Rust, C++ references).
- **Aliasing** — two references to the same data. Aliasing + mutation = bugs. Rust prevents this: *either* multiple immutable references *or* one mutable reference, never both.

**Memory management strategies**:

- **Garbage collection** (Java, Go, JS) — runtime frees unreachable objects. Convenient, but GC pauses.
- **Manual** (C, C++) — developer calls `free()`/`delete`. No pauses, but leaks/double-free/use-after-free.
- **Ownership** (Rust) — compiler tracks lifetimes. Freed when owner goes out of scope. No GC, no manual free.

---

## 4. Value vs Reference Semantics; Copy vs Move

When you assign `b = a`, does `b` get a copy or a reference to the same data? This determines whether modifying `b` also changes `a`.

- **Value semantics** — assignment copies data. Go structs, Rust primitives. `b.x = 99` doesn't affect `a`.
- **Reference semantics** — assignment copies a pointer. JS objects, Java objects. `b.x = 99` changes `a.x` too.
- **Move semantics** — ownership transfers, original invalidated. Rust's default for heap data.

```javascript
// JS reference gotcha:
const a = { x: 1 };
const b = a;       // same object
b.x = 99;          // a.x is now 99!
```

```rust
// Rust move:
let a = String::from("hello");
let b = a;          // a is MOVED — using a after this is a compile error
```

**Copy vs clone**: Copy is cheap (bitwise, for primitives). Clone is deep copy (explicit, potentially expensive). In JS, spread `{...obj}` is shallow — use `structuredClone()` for deep copies.

---

## 5. Closures, Higher-Order Functions, Recursion

- **Higher-order functions** — take or return functions. `map`, `filter`, `reduce` are the canonical examples:

```typescript
const nums = [1, 2, 3, 4, 5];
nums.map(n => n * 2);              // [2, 4, 6, 8, 10]
nums.filter(n => n % 2 === 0);      // [2, 4]
nums.reduce((acc, n) => acc + n, 0); // 15
```

- **Closures** — functions that capture variables from their enclosing scope:

```typescript
function makeCounter() {
  let count = 0;                    // captured by the closure
  return () => { count++; return count; };
}
const counter = makeCounter();
counter(); // 1
counter(); // 2 — count persists because the closure holds a reference
```

- **Recursion** — a function calling itself. Needs a base case and a recursive case that makes progress. Maps naturally to recursive data structures (trees, nested JSON). **Tail call optimization (TCO)** reuses the stack frame if the recursive call is the last operation — supported in Safari, not in V8.

---

## 6. Polymorphism: Parametric vs Subtype; Generics

Polymorphism lets you write code that works with multiple types instead of writing `sortInts`, `sortStrings`, `sortDates` separately.

- **Subtype polymorphism** — accept a base type, work with any subtype. Uses dynamic dispatch (vtable) at runtime. Flexible but has overhead.
- **Parametric polymorphism (generics)** — parameterized by a type variable. No runtime cost (erased in TS, monomorphized in Rust/C++).
- **Bounded polymorphism** — generics with constraints.

```typescript
// Subtype: any Shape works
interface Shape { area(): number; }
function totalArea(shapes: Shape[]): number {
  return shapes.reduce((sum, s) => sum + s.area(), 0);
}

// Generics: works for any T
function first<T>(arr: T[]): T | undefined { return arr[0]; }

// Bounded: T must have length
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
```

---

## 7. Error Models: Exceptions, Result Types, Panic/Abort

- **Exceptions** (Java, Python, JS) — errors jump up the call stack to the nearest `catch`. Convenient but invisible in function signatures. Easy to forget to handle.
- **Result types** (Rust `Result<T, E>`, Go `(value, error)`) — errors are return values. Explicit, forces handling, no hidden control flow. More verbose.
- **Panic/abort** — unrecoverable errors (out of memory, violated invariants).

```typescript
// Exceptions: caller must know to catch — nothing in the signature says so
function parseJSON(input: string): any {
  return JSON.parse(input);  // throws on invalid JSON
}

// Result-style: explicit in the return type
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
function safeParse(input: string): Result<any, string> {
  try { return { ok: true, value: JSON.parse(input) }; }
  catch (e) { return { ok: false, error: String(e) }; }
}
```

**Rule of thumb**: exceptions for truly exceptional situations (programmer errors). Result types for expected failures (invalid input, network timeouts).

---

## 8. Concurrency Models

- **Threads + shared memory** — multiple threads share an address space, coordinate via locks/mutexes. Powerful but error-prone (races, deadlocks). Best for CPU-bound work.
- **Async/await** — one thread handles many tasks by suspending/resuming. No parallelism in a single-threaded loop, but excellent for I/O-bound work. JavaScript, Python asyncio, Rust async.
- **Actors/channels** — concurrency via message passing. Each actor has private state. Erlang/Elixir, Go channels. Eliminates data races by design.

Async runtimes (Tokio, Node worker threads) can add parallelism on top of async.

---

## 9. Determinism, Side Effects, Purity, Referential Transparency

- **Pure function** — same inputs always produce the same output, no side effects. `Math.max(3, 5)` is pure. `Date.now()` is not.
- **Side effects** — anything beyond returning a value: I/O, mutation, network calls, logging.
- **Referential transparency** — an expression can be replaced with its value without changing behavior.

Pure functions are easier to test, parallelize, cache, and reason about. **Push side effects to the edges** — keep business logic pure, handle I/O at the boundary:

```typescript
// Impure: side effects mixed with logic
function processOrder(orderId: string) {
  const order = db.getOrder(orderId);    // DB read
  const total = calculateTotal(order);    // pure logic — isolate this
  db.updateOrder(orderId, { total });     // DB write
}
// Better: pure core, impure shell
function calculateTotal(items: Item[]): number {  // pure, testable
  return items.reduce((sum, i) => sum + i.price * i.qty, 0);
}
```

---

## 10. TypeScript/JavaScript-Specific Gotchas

### The event loop

JavaScript is single-threaded with an event loop. The loop processes a queue of tasks:

1. Execute the current synchronous code (the **call stack**)
2. Process all **microtasks** (Promise callbacks, `queueMicrotask`)
3. Process one **macrotask** (`setTimeout`, `setInterval`, I/O callbacks)
4. Repeat

```typescript
console.log("1");                    // Sync
setTimeout(() => console.log("2"), 0); // Macrotask
Promise.resolve().then(() => console.log("3")); // Microtask
console.log("4");                    // Sync

// Output: 1, 4, 3, 2
// Why: sync runs first, then microtasks (Promise), then macrotasks (setTimeout)
```

**Key implication**: a long-running synchronous operation blocks everything — no I/O, no timers, no promise callbacks. This is why `while(true)` freezes a browser tab.

### Prototype chain and other gotchas

JavaScript is prototype-based (`class` is syntactic sugar). Every object has a `[[Prototype]]` link — property lookup walks the chain. `instanceof` walks it too.

- **`==` vs `===`** — always use `===`. Loose equality has bizarre coercion (`[] == false` is `true`).
- **`this` binding** — depends on *how* a function is called. Arrow functions capture `this` from their enclosing scope; regular functions don't.
- **Floating point** — `0.1 + 0.2 !== 0.3` (IEEE 754, not JS-specific).
- **`typeof null === "object"`** — a 30-year-old bug that can never be fixed.

---

## Exercises

1. **Types**: Give an example where TypeScript's `any` lets code compile but crash at runtime. Fix it using `unknown` and type narrowing.
2. **Memory**: Explain why `let a = vec![1,2,3]; let b = a; println!("{:?}", a);` won't compile in Rust. What concept does the compiler enforce?
3. **Closures**: Write a `memoize` function in TypeScript that caches results of a single-argument function using a closure.
4. **Event loop**: Predict the output of `async function run() { console.log("A"); await Promise.resolve(); console.log("B"); } run(); console.log("C");` — and explain why.
5. **Error models**: Refactor a `try/catch` function to return a `Result` type. When is each approach more appropriate?
6. **Value vs reference**: Explain the bug: `const config = { retries: 3 }; const local = config; local.retries = 0;`. How do you fix it?

---

## Recommended Resources

- **Book**: *Crafting Interpreters* by Robert Nystrom (free online at craftinginterpreters.com) — build two interpreters from scratch; covers compilation, bytecode, VMs, GC
- **Book**: *Types and Programming Languages* by Benjamin Pierce — the academic reference on type systems
- **Book**: *Programming Rust* by Blandy, Orendorff, Tindall — best way to deeply understand ownership, borrowing, and memory safety
- **Article**: "What Color is Your Function?" by Bob Nystrom — excellent essay on async/sync function coloring
- **Video**: Philip Roberts "What the heck is the event loop anyway?" (JSConf) — the definitive event loop explanation
- **Interactive**: TypeScript Playground (typescriptlang.org/play) — experiment with types, generics, and inference
- **Reference**: MDN Web Docs — the best JavaScript/TypeScript reference for event loop, prototypes, and runtime behavior

---

*Previous lesson: [Computer Architecture](./02_computer_architecture.md) | Next: Module 2 — [Data Structures](../module_02_dsa/01_data_structures.md)*
