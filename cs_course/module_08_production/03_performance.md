# Performance

Performance work is counterintuitive: the biggest gains almost never come from where you think they will. The developer who rewrites a function in a "faster" language often misses the N+1 query adding 500ms to every page load. The first rule of performance engineering is **measure first** — then optimize the thing that actually matters. This lesson covers the tools, techniques, and mental models for systematic performance work.

---

## Measure First

### Why Intuition Fails

Developers are notoriously bad at guessing where performance problems live. Code that *looks* slow (complex logic, many lines) often runs in microseconds. Code that *looks* fine (a simple database call inside a loop) can take seconds. Without measurement, you'll optimize the wrong thing and waste time while the real bottleneck remains untouched.

### Profiling

A profiler instruments your code and records where time is spent:

```python
# Python: cProfile
import cProfile

cProfile.run('process_orders()')
# Output: shows time spent in each function, sorted by cumulative time
#   ncalls  tottime  cumtime  function
#   1       0.001    3.412    process_orders
#   250     0.002    3.210    fetch_order_details   ← 94% of time here!
#   250     3.200    3.200    {method 'execute' of cursor}
```

```bash
# Node.js: built-in profiler
node --prof app.js
node --prof-process isolate-*.log > profile.txt

# Go: pprof
import _ "net/http/pprof"
# Then: go tool pprof http://localhost:6060/debug/pprof/profile
```

### Flame Graphs

Flame graphs visualize profiler output as stacked horizontal bars. The x-axis is **not** time — it's the alphabetical sort of function names. The y-axis is stack depth. **Width represents total time spent in that function and its children.**

```
|------- main() ----------------------------------------|
|--- handleRequest() ----|--- processPayment() ---------|
|-- parseJSON() --|      |-- validateCard() --|-- db() --|
```

Reading a flame graph:
- **Wide bars at the top** = functions that directly consume lots of CPU
- **Wide bars at the bottom** = functions called frequently or with slow children
- **Narrow towers** = deep call stacks that are fast (not a problem)
- **Plateaus** = time spent in one function, likely your bottleneck

**Tools**: `perf` + Brendan Gregg's FlameGraph scripts (Linux), Chrome DevTools (JS), `py-spy` (Python), `async-profiler` (Java), `pprof` (Go).

---

## Latency Budget

Every user-visible operation has a latency budget — the total time from click to response. Break it down to find what you can optimize:

```
User clicks "Place Order"
├─ Client-side JS processing:    20ms
├─ Network (client → server):    80ms   (geographic distance, TLS)
├─ Server processing:            50ms
│  ├─ Auth middleware:           5ms
│  ├─ Business logic:            10ms
│  └─ Database queries:          35ms
├─ External API (payment):       150ms   ← biggest chunk
├─ Network (server → client):    80ms
└─ Client-side rendering:        30ms
   Total:                        410ms
```

**Insights from budgeting**:
- The payment API takes 150ms — can you call it asynchronously?
- Network is 160ms round-trip — serving from a closer region could halve this
- Database is 35ms — probably fine, but check for N+1 queries
- Client rendering is 30ms — could improve with lazy loading

**Key principle**: optimize the largest slice first. Shaving 2ms off business logic doesn't matter when the payment API takes 150ms.

---

## Caching Strategy and Invalidation

Caching is the most powerful performance tool — and cache invalidation is one of the hardest problems in computer science.

### Cache Layers

```
User → CDN (edge cache) → Reverse Proxy (Nginx) → App Cache (Redis) → Database
         10ms                    20ms                  5ms               50ms
```

Each layer catches requests earlier, avoiding expensive downstream work:

- **CDN / Edge cache**: static assets, API responses with `Cache-Control` headers. Serves from the nearest PoP
- **Reverse proxy cache**: full page or API response caching at the server level
- **Application cache (Redis/Memcached)**: computed results, session data, frequently-read objects
- **Database query cache**: query result caching (use with caution — most databases do this poorly)
- **In-process cache**: data kept in application memory. Fastest, but doesn't share across instances

### Cache Invalidation Strategies

The hard part isn't caching — it's knowing **when the cache is wrong**:

**Time-based (TTL)**:
```python
cache.set("user:123", user_data, ttl=300)  # expires after 5 minutes
```
Simple and predictable. Trade-off: users see stale data for up to TTL seconds.

**Event-based (write-through / write-behind)**:
```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")   # invalidate on write
```
More consistent, but requires discipline — every write path must invalidate.

**Cache-aside (lazy loading)**:
```python
def get_user(user_id):
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    cache.set(f"user:{user_id}", user, ttl=300)
    return user
```

### Common Caching Pitfalls

- **Cache stampede**: cache expires, 1000 concurrent requests all hit the database simultaneously. Solution: lock on cache miss (only one request regenerates), or stagger TTLs with jitter
- **Stale data**: user updates their profile but sees old data. Solution: invalidate on write, or use short TTLs for user-facing data
- **Memory pressure**: caching everything fills up memory. Use LRU eviction and monitor hit rates. A cache with a 30% hit rate is wasting memory
- **Caching errors**: accidentally caching a 500 error response. Always check response validity before caching

---

## N+1 Queries, Batching, and Pagination

### The N+1 Problem

One of the most common performance bugs in web applications:

```python
# N+1: 1 query for orders + N queries for users (one per order)
orders = db.query("SELECT * FROM orders LIMIT 100")  # 1 query
for order in orders:
    user = db.query(f"SELECT * FROM users WHERE id = {order.user_id}")  # 100 queries!
```

**Fix with a JOIN or batch query:**

```python
# JOIN: 1 query total
results = db.query("""
    SELECT o.*, u.name, u.email
    FROM orders o
    JOIN users u ON o.user_id = u.id
    LIMIT 100
""")

# Batch: 2 queries total
orders = db.query("SELECT * FROM orders LIMIT 100")
user_ids = [o.user_id for o in orders]
users = db.query(f"SELECT * FROM users WHERE id IN ({','.join(user_ids)})")
```

**In ORMs**: use eager loading (`include`, `prefetch_related`, `joinedload`) instead of lazy loading for relationships you know you'll access.

### Batching

Group multiple operations into one:

```python
# Bad: 1000 individual inserts
for item in items:
    db.execute("INSERT INTO items VALUES (%s, %s)", (item.id, item.name))

# Good: 1 batch insert
db.executemany("INSERT INTO items VALUES (%s, %s)",
               [(item.id, item.name) for item in items])

# Even better: COPY (Postgres) or LOAD DATA (MySQL) for bulk imports
```

### Pagination

Never return unbounded result sets:

```sql
-- Offset-based (simple, but slow for deep pages)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 1000;
-- Problem: DB must scan and discard 1000 rows

-- Cursor-based / keyset pagination (fast for any page depth)
SELECT * FROM orders
WHERE created_at < '2025-01-15T10:30:00Z'
ORDER BY created_at DESC
LIMIT 20;
-- Only scans 20 rows, regardless of position
```

**Rule of thumb**: use offset pagination for small datasets or admin UIs. Use cursor-based pagination for user-facing feeds, infinite scroll, or any dataset that could grow large.

---

## Memory Leaks and GC Pauses

### Memory Leaks

Memory that is allocated but never freed, causing usage to grow over time until the process crashes or the system swaps to disk.

**Common causes**:
- **Unbounded caches**: maps or arrays that grow without eviction
- **Event listener leaks**: adding listeners without removing them (common in Node.js and browser JS)
- **Closures capturing large objects**: a closure retains a reference to variables in its scope, keeping them alive
- **Global state accumulation**: appending to global arrays, growing connection pools without limits

```javascript
// Memory leak: listeners added on every request, never removed
app.get('/stream', (req, res) => {
  const handler = (data) => res.write(data);
  eventEmitter.on('data', handler);
  // If the client disconnects, handler is never removed!

  // Fix: clean up on disconnect
  req.on('close', () => eventEmitter.removeListener('data', handler));
});
```

**Detection**: monitor RSS (Resident Set Size) over time. If it trends upward continuously without stabilizing, you have a leak. Use heap snapshots to identify what's growing.

### GC Pauses

Garbage-collected languages (Java, Go, C#, JavaScript) periodically pause to reclaim unused memory. These pauses directly impact tail latency.

- **Stop-the-world GC**: all application threads pause during collection. Older Java GC implementations could pause for hundreds of milliseconds
- **Concurrent/incremental GC**: collection happens alongside application work, with shorter pauses. Modern collectors (Go's GC, Java's ZGC/Shenandoah) target sub-millisecond pauses
- **Generational hypothesis**: most objects die young. Generational GCs exploit this by collecting "young generation" frequently (fast, small pauses) and "old generation" rarely (slower, larger pauses)

**Mitigation**:
- Reduce allocation rate: reuse objects, use object pools for hot paths
- Right-size heap: too small = frequent GC; too large = longer pauses when GC does run
- Choose the right collector: for latency-sensitive services, use a low-pause collector (ZGC, Shenandoah for Java; Go's GC is already excellent)

---

## Tail Latency

### Why p95/p99 Matters More Than Average

If your average latency is 50ms but your p99 is 2 seconds, one in every 100 requests takes 40x longer than average. At scale, this means:

- A page that makes 20 API calls: the probability that *at least one* hits p99 is `1 - (0.99)^20 = 18%`. Nearly 1 in 5 page loads is slow
- A service that fans out to 50 backends: `1 - (0.99)^50 = 39%`. Over a third of requests are slow

**The more services you call, the more tail latency dominates user experience.**

```
# Latency distribution — averages hide the pain
p50  (median):  45ms    ← "typical" request
p90:            120ms
p95:            350ms   ← 1 in 20 requests
p99:            1800ms  ← 1 in 100 requests — 40x the median
p99.9:          5200ms  ← 1 in 1000 — usually timeouts or retries
```

### Common Causes of Tail Latency

- **GC pauses**: stop-the-world collection hits the unlucky requests
- **Lock contention**: threads waiting for a mutex add latency for some requests
- **Cold caches**: first request after cache expiry hits the database
- **Background tasks**: a cron job or compaction competing for CPU/disk
- **Noisy neighbors**: shared infrastructure (VMs, containers) where another tenant spikes
- **Retries**: a timeout + retry doubles the latency for that request

### Mitigation Strategies

- **Hedged requests**: send the same request to two replicas, use whichever responds first. Effective but doubles load
- **Timeouts**: don't wait forever for a slow dependency. Fail fast and degrade
- **Request deadlines**: propagate a deadline through the call chain. If the deadline has passed, don't start work — the caller has already given up
- **Load shedding**: under overload, reject new requests early (503) rather than serving all requests slowly
- **Isolate noisy operations**: run batch jobs, compaction, and analytics on separate instances from latency-sensitive traffic

---

## Exercises

1. **Profile a hot path**: pick a frequently-called function or endpoint in a project you work on. Profile it, generate a flame graph, and identify the top 3 time consumers. Was your intuition about the bottleneck correct?

2. **Latency budget breakdown**: pick a user-facing operation in your application. Break its latency into components (client, network, server, database, external services). Draw the budget. Where's the biggest opportunity?

3. **N+1 hunting**: review a data-fetching layer in your codebase (API endpoint, GraphQL resolver, background job). Look for N+1 patterns. Fix one with a JOIN or batch query and measure the difference.

4. **Cache hit rate analysis**: if your application uses a cache (Redis, Memcached, in-memory), measure the hit rate. What's being cached? Is anything cached that shouldn't be? Is anything *not* cached that should be?

5. **Tail latency investigation**: collect p50, p90, p95, and p99 latency for a service you run. Calculate the ratio of p99 to p50. If it's greater than 10x, investigate: is it GC, database, lock contention, or something else? Propose a mitigation.

---

## Recommended Resources

- **"Systems Performance" by Brendan Gregg** — the definitive reference on performance analysis. Covers everything from CPU to disk to network with practical tools and methodology.
- **Brendan Gregg's website ([brendangregg.com](https://www.brendangregg.com))** — flame graphs, USE method, and dozens of performance analysis tools and techniques. Essential reading.
- **"High Performance Browser Networking" by Ilya Grigorik (free online: [hpbn.co](https://hpbn.co))** — network performance from the browser perspective: TCP, TLS, HTTP/2, WebSocket.
- **"The Tail at Scale" by Jeff Dean and Luiz Barroso (paper)** — seminal Google paper on why tail latency matters and techniques to mitigate it.
- **"Designing for Performance" by Martin Thompson** — mechanical sympathy, cache-friendly data structures, and low-latency design principles.
- **USE Method documentation ([brendangregg.com/usemethod.html](https://www.brendangregg.com/usemethod.html))** — a systematic approach to performance analysis that prevents you from missing bottlenecks.
