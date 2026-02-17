# Testing and Verification

Software that hasn't been tested is software you're guessing about. Testing isn't just about catching bugs — it's about building confidence that your system behaves correctly under the conditions that matter. The real skill is knowing *what* to test, *how much* to test, and choosing the right techniques for each situation. This lesson covers the full spectrum: from unit tests to formal methods, from property-based testing to static analysis.

---

## Why Testing Matters

Bugs found in production are 10–100x more expensive to fix than bugs found during development. But the cost isn't just in engineering time — it's in user trust, incident response, data corruption, and security breaches. Good testing strategy:

- **Catches regressions** before they reach users
- **Documents behavior** — tests are executable specifications
- **Enables refactoring** — you can change code with confidence when tests exist
- **Reduces cognitive load** — you don't have to hold the entire system in your head

The goal isn't 100% code coverage. The goal is **high confidence that the system works correctly in the scenarios that matter**.

---

## Test Types

### Unit Tests

Test individual functions or classes in isolation. Dependencies are mocked or stubbed.

```python
# Function under test
def calculate_discount(price: float, tier: str) -> float:
    if tier == "premium":
        return price * 0.20
    elif tier == "standard":
        return price * 0.10
    return 0.0

# Unit test
def test_premium_discount():
    assert calculate_discount(100.0, "premium") == 20.0

def test_unknown_tier_no_discount():
    assert calculate_discount(100.0, "unknown") == 0.0
```

**Strengths**: fast (milliseconds), easy to write, pinpoint failures precisely.
**Weaknesses**: don't verify that components work together. Over-mocking can test the mocks instead of the code.

### Integration Tests

Test that multiple components work together correctly — a service with its real database, an API endpoint with its middleware, two services communicating over HTTP.

```python
# Integration test: API + database together
def test_create_and_retrieve_user(test_client, test_db):
    response = test_client.post("/users", json={"name": "Ada", "email": "ada@example.com"})
    assert response.status_code == 201

    user_id = response.json()["id"]
    response = test_client.get(f"/users/{user_id}")
    assert response.json()["name"] == "Ada"
```

**Strengths**: catch wiring bugs, serialization issues, database constraint problems.
**Weaknesses**: slower, require infrastructure (databases, queues), harder to debug failures.

### End-to-End (E2E) Tests

Test the full system from the user's perspective — typically by driving a browser or hitting the public API with no mocks.

```javascript
// E2E test with Playwright
test('user can sign up and see dashboard', async ({ page }) => {
  await page.goto('/signup');
  await page.fill('#email', 'test@example.com');
  await page.fill('#password', 'securepassword123');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Welcome');
});
```

**Strengths**: highest confidence that the real system works for real users.
**Weaknesses**: slow, flaky (timing issues, network), expensive to maintain, hard to debug. Use sparingly for critical user journeys.

### Contract Tests

Verify that services agree on the shape of their communication — without running both services simultaneously. Essential in microservice architectures.

```python
# Provider side: "I promise my /users endpoint returns this shape"
# Consumer side: "I expect the /users endpoint to return this shape"
# Contract test: verify both sides agree

# Example with Pact-style contract
expected_response = {
    "id": Like(1),                    # any integer
    "name": Like("string"),           # any string
    "email": Like("user@example.com") # any string matching pattern
}
```

**Why they matter**: if Service A depends on Service B's API, and Service B changes its response format, a contract test catches the break *before* deployment — without needing both services running.

---

## Property-Based Testing

Instead of testing specific examples, property-based testing generates hundreds of random inputs and verifies that **invariants** (properties) always hold.

```python
from hypothesis import given
import hypothesis.strategies as st

@given(st.lists(st.integers()))
def test_sort_preserves_length(xs):
    assert len(sorted(xs)) == len(xs)

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)

@given(st.lists(st.integers(), min_size=1))
def test_sort_first_element_is_minimum(xs):
    assert sorted(xs)[0] == min(xs)
```

**Key idea**: instead of saying "sort([3,1,2]) == [1,2,3]", you say "for *any* list, sorting preserves length, is idempotent, and puts the minimum first." The framework generates edge cases you'd never think of: empty lists, single elements, duplicates, huge numbers, negative numbers.

**When to use**: serialization/deserialization roundtrips, parsers, data transformations, anything with clear mathematical properties.

**Tools**: Hypothesis (Python), fast-check (JS/TS), QuickCheck (Haskell/Erlang), PropEr (Erlang).

---

## Fuzzing

Fuzzing throws massive amounts of random, malformed, or unexpected input at your code to find crashes, hangs, and security vulnerabilities. It's "property-based testing's chaotic cousin."

```bash
# AFL (American Fuzzy Lop) — a coverage-guided fuzzer
afl-fuzz -i input_corpus/ -o findings/ -- ./my_parser @@

# libFuzzer — integrated into the compilation
# Write a fuzz target:
```

```c
// C fuzz target for libFuzzer
int LLVMFuzzerTestEntry(const uint8_t *data, size_t size) {
    parse_input(data, size);  // if this crashes, fuzzer found a bug
    return 0;
}
```

**How it works**: coverage-guided fuzzers instrument your code and track which branches are taken. They mutate inputs to explore new code paths, finding inputs that trigger crashes, assertions, or undefined behavior.

**When to use**: parsers, deserializers, network protocol handlers, file format readers — anywhere untrusted input is processed.

**Real-world impact**: fuzzing has found thousands of bugs in Chrome, Firefox, the Linux kernel, OpenSSL, and countless libraries. Google's OSS-Fuzz continuously fuzzes 1,000+ open-source projects.

---

## Static Analysis and Type Systems

These are "tests that run without executing your code." They catch entire categories of bugs at build time.

### Linters

Enforce style rules and catch common mistakes:

```javascript
// ESLint catches: unused variables, unreachable code, == vs ===
if (x == null) { }   // warning: use === for strict comparison
```

### Type Systems as Cheap Tests

A strong type system prevents bugs that would otherwise need tests:

```typescript
// Without types: need tests to verify you don't pass wrong arguments
function createUser(name, email, age) { ... }
createUser(25, "alice", "alice@example.com"); // runtime error, args swapped

// With types: the compiler catches this at build time
function createUser(name: string, email: string, age: number): User { ... }
createUser(25, "alice", "alice@example.com"); // compile error!
```

**Exhaustiveness checking** — the compiler ensures you handle all cases:

```typescript
type PaymentStatus = "pending" | "completed" | "failed" | "refunded";

function handlePayment(status: PaymentStatus): string {
    switch (status) {
        case "pending": return "Waiting...";
        case "completed": return "Done!";
        case "failed": return "Error!";
        // Compiler error: "refunded" not handled — no test needed
    }
}
```

### Static Analysis Tools

Go beyond linting — analyze data flow, find null dereferences, detect security issues:

- **Semgrep**: pattern-based analysis with custom rules
- **CodeQL**: query-based analysis (GitHub uses it for security scanning)
- **mypy / Pyright**: Python type checkers
- **SonarQube**: multi-language analysis with quality gates

**Key insight**: static analysis, linters, and types are the cheapest "tests" you can run. They're fast, deterministic, and catch bugs before code is even committed. Maximize their use before writing runtime tests.

---

## Formal Methods

When correctness really matters — distributed protocols, consensus algorithms, financial systems, safety-critical software — informal testing isn't enough.

### Model Checking

Define your system as a state machine, specify the properties it must satisfy, and let the tool exhaustively explore every possible state.

**TLA+** (Temporal Logic of Actions) is the most accessible formal specification language for distributed systems:

```
\* TLA+ specification for a simple counter with a limit
VARIABLE counter

Init == counter = 0

Increment == counter < 10 /\ counter' = counter + 1

Spec == Init /\ [][Increment]_counter

\* Safety property: counter never exceeds 10
SafetyInvariant == counter <= 10
```

The TLC model checker will explore every reachable state and verify that `SafetyInvariant` always holds. If it doesn't, TLC produces a counterexample trace showing exactly how the violation occurs.

**Real-world use**: Amazon used TLA+ to find subtle bugs in DynamoDB, S3, and EBS that testing alone could not have caught. These bugs involved rare interleavings of concurrent operations.

### When to Use Formal Methods

- Distributed consensus protocols (Raft, Paxos)
- Concurrent data structures
- Financial transaction processing
- Safety-critical systems (aviation, medical devices)
- Any system where "almost correct" isn't good enough

### When NOT to Use Them

- CRUD applications
- UI code
- Most business logic
- Anywhere the cost of formal specification outweighs the cost of a bug

---

## The Test Pyramid (and Why It's Context-Dependent)

The classic test pyramid suggests: many unit tests, fewer integration tests, even fewer E2E tests.

```
        /  E2E  \          <- few, slow, high confidence
       /----------\
      / Integration \      <- moderate count
     /----------------\
    /    Unit Tests     \  <- many, fast, focused
```

**But this is a guideline, not a law.** The right mix depends on your system:

- **Library/algorithm-heavy code**: unit tests dominate. The pyramid works well.
- **Glue code / API services**: integration tests matter most. Business logic is in how components connect, not in isolated functions. The pyramid inverts to a "trophy" or "honeycomb" shape.
- **UI-heavy applications**: E2E tests on critical paths + component tests provide the most value.

**What matters is reliability, not ideology.** A test suite with 95% unit test coverage that misses every integration bug is worse than a smaller suite with well-chosen integration tests. Optimize for catching the bugs that *actually happen in your system*.

### Testing Anti-Patterns to Avoid

- **Testing implementation details**: tests that break when you refactor but not when behavior changes
- **Excessive mocking**: if your test mocks 8 dependencies, it's not testing much
- **Flaky tests**: a test that sometimes passes and sometimes fails erodes trust in the entire suite. Fix or delete it
- **Slow tests blocking development**: if your test suite takes 30 minutes, developers stop running it
- **No tests at the boundaries**: the most bugs live at system boundaries — API endpoints, database queries, external service calls

---

## Exercises

1. **Testing strategy design**: you're building an e-commerce checkout service that calls a payment gateway, updates inventory, and sends a confirmation email. Design a testing strategy: what do you unit test, integration test, and E2E test? What do you mock?

2. **Property-based testing**: pick a function you've written recently (a parser, a data transformation, a validator). Identify 3 invariants it should satisfy. Write property-based tests for them using Hypothesis or fast-check.

3. **Type system exercise**: take a function with `any` types (or untyped JavaScript) and add strict types. Identify at least 2 bugs or edge cases the type system now prevents without any runtime tests.

4. **Fuzzing exploration**: find a simple parser or deserializer in an open-source project. Read about how fuzzing has been applied to it (many projects document their fuzzing setup). What kinds of bugs were found?

5. **Test anti-pattern audit**: review a test file in a project you work on. Identify any anti-patterns: excessive mocking, testing implementation details, flaky behavior, or missing boundary tests. Propose improvements.

---

## Recommended Resources

- **"Unit Testing Principles, Practices, and Patterns" by Vladimir Khorikov** — the best book on *what* to test and how to write tests that actually catch bugs without being brittle.
- **Hypothesis documentation ([hypothesis.readthedocs.io](https://hypothesis.readthedocs.io))** — excellent introduction to property-based testing with practical examples.
- **"Lessons from Formal Verification Applied at Amazon" (paper)** — how AWS uses TLA+ in production. Eye-opening for the value of formal methods.
- **Google Testing Blog** — practical advice on testing strategy, flaky tests, and test infrastructure at scale.
- **"Fuzzing Book" ([fuzzingbook.org](https://www.fuzzingbook.org))** — interactive online book covering fuzzing techniques from basic to advanced.
- **Lamport's TLA+ video course** — Leslie Lamport himself teaches TLA+. Dense but invaluable if you work on distributed systems.
