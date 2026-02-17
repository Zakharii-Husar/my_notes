# Systems Design Patterns

Knowing distributed systems theory is essential, but shipping a reliable product requires choosing the right architecture and applying the right patterns. This lesson covers the building blocks that appear in almost every modern backend: how to structure services, how to move data between them, how to cache, how to search, and how to store different kinds of data. These aren't abstract — they're the decisions you'll make (or inherit) on every team.

---

## Service Architecture

### Monolith

A single deployable unit containing all functionality. Despite the industry's microservices enthusiasm, monoliths are the right starting point for most teams.

**Advantages**:
- Simple to develop, test, deploy, and debug
- No network calls between components — just function calls
- Single database, single transaction boundary
- Easy to refactor: move code between modules with your IDE

**When it breaks down**:
- Team grows beyond ~15-20 engineers stepping on each other
- Deploy cycle slows because unrelated changes are coupled
- One component's resource needs dominate (e.g., video processing starves the API)

### Modular Monolith

A monolith with **explicit module boundaries** — the best of both worlds for most organizations:

```
┌──────────────────────────────────────────┐
│              Monolith Process             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  Orders  │ │ Payments │ │ Shipping │ │
│  │  Module  │ │  Module  │ │  Module  │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       │             │             │       │
│       └─────── Public APIs ──────┘       │
│              (no direct DB access         │
│               across modules)             │
└──────────────────────────────────────────┘
```

- Each module has its own public API (interfaces/classes), internal logic, and (ideally) its own database schema or tables
- Modules communicate only through defined interfaces — no reaching into another module's tables
- You get most microservice benefits (clear boundaries, independent teams per module) without the operational complexity
- Extraction to a separate service later is straightforward because the boundary already exists

### Microservices

Each service is independently deployable, has its own data store, and communicates over the network.

**Genuine benefits**:
- Independent deployment: ship the payments service without touching orders
- Independent scaling: scale the search service separately from the user service
- Technology heterogeneity: use Python for ML, Go for the API gateway, Java for the billing system

**Costs that are often underestimated**:
- Network calls replace function calls (latency, failure modes, serialization)
- Distributed transactions are hard (see: saga pattern, eventual consistency)
- Operational complexity: service discovery, observability across services, deployment orchestration
- Data consistency: no more JOINs across service boundaries
- Testing: integration tests require running multiple services

**Rule of thumb**: start with a modular monolith. Extract a microservice when you have a *specific* reason (independent scaling, different deployment cadence, different team ownership). Never extract "just in case."

---

## API Patterns

### API Gateway

A single entry point that sits in front of your backend services:

```
Client ──→ API Gateway ──→ Service A
                       ──→ Service B
                       ──→ Service C
```

**Responsibilities**:
- **Routing**: direct requests to the right service based on path/headers
- **Authentication**: verify tokens once at the gateway, pass identity downstream
- **Rate limiting**: protect backend services from abuse
- **Request aggregation**: combine responses from multiple services into one client response
- **Protocol translation**: accept REST from clients, use gRPC internally

**Watch out for**: the gateway becoming a monolith itself. Keep it thin — routing, auth, rate limiting. Business logic belongs in services.

### Backend for Frontend (BFF)

A specialized API layer tailored to a specific client type:

```
Mobile App  ──→  Mobile BFF  ──→  Services
Web App     ──→  Web BFF     ──→  Services
Admin Panel ──→  Admin BFF   ──→  Services
```

**Why**: mobile clients need compact payloads and fewer round trips. Web apps need different data shapes. An admin dashboard needs batch operations. A single generic API serves none of them well.

**Each BFF**:
- Aggregates data from multiple services into the shape its client needs
- Handles client-specific concerns (pagination style, field selection, image sizing)
- Is owned by the client team, not the backend team

### Service Mesh

An infrastructure layer that handles service-to-service communication:

```
┌─────────────┐         ┌─────────────┐
│  Service A  │         │  Service B  │
│  ┌───────┐  │  mTLS   │  ┌───────┐  │
│  │ Proxy  │◄─┼────────┼─►│ Proxy  │  │
│  │(sidecar)│ │         │  │(sidecar)│ │
│  └───────┘  │         │  └───────┘  │
└─────────────┘         └─────────────┘
```

- A sidecar proxy (e.g., Envoy) runs alongside each service
- Handles: mTLS (mutual TLS), retries, circuit breaking, load balancing, observability (tracing headers)
- **When**: you have many services and cross-cutting networking concerns. A mesh lets you apply policies uniformly without modifying each service
- **When NOT**: 5 services don't need a mesh. The operational complexity of running Istio/Linkerd is significant. Start with library-level solutions (retry logic in your HTTP client)

---

## CQRS and Event Sourcing

### CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries):

```
                    ┌──────────────┐
  Commands ──────→  │  Write Model │ ──→ Event/Change ──→ ┌────────────┐
  (create, update)  │  (normalized)│                      │ Read Model │
                    └──────────────┘                      │(denormalized│
  Queries ──────────────────────────────────────────────→ │  for reads) │
                                                          └────────────┘
```

**Where it helps**:
- Read and write patterns are very different (e.g., complex domain writes, simple list/search reads)
- Read and write loads need to scale independently
- You need multiple read representations (e.g., a search index, a reporting view, a cache)

**Where it hurts**:
- Adds complexity: now you maintain two models and synchronization between them
- Read model is eventually consistent with the write model (there's a propagation delay)
- Overkill for simple CRUD applications

### Event Sourcing

Instead of storing current state, store a sequence of events that led to the current state:

```
Traditional:   Account { balance: 150 }

Event sourced:
  1. AccountCreated { initial_balance: 100 }
  2. MoneyDeposited { amount: 200 }
  3. MoneyWithdrawn { amount: 150 }
  Current state = replay events → balance: 150
```

**Benefits**:
- Complete audit trail — you know exactly how you got to the current state
- Can rebuild state at any point in time (debugging, compliance)
- Can derive new read models from the event log retroactively
- Natural fit with CQRS (events feed the read model)

**Costs**:
- Event schema evolution is tricky (what happens when the event format changes?)
- Rebuilding state from millions of events is slow → need **snapshots**
- Querying is complex: you can't just `SELECT * WHERE balance > 100`
- Not every domain benefits. Don't use event sourcing for a blog

---

## Async Messaging

When services communicate asynchronously, they decouple in time — the sender doesn't wait for the receiver.

### Message Queues

Point-to-point: one producer, one consumer (or competing consumers from a pool):

```
Producer ──→ [ Queue ] ──→ Consumer
                       ──→ Consumer  (competing, load-balanced)
```

- **Semantics**: each message is processed by exactly one consumer
- **Use case**: work distribution (process uploaded images, send emails)
- **Examples**: RabbitMQ, Amazon SQS, Sidekiq (Redis-backed)

### Pub/Sub (Publish-Subscribe)

One publisher, multiple independent subscribers:

```
Publisher ──→ [ Topic ] ──→ Subscriber A (email service)
                        ──→ Subscriber B (analytics service)
                        ──→ Subscriber C (audit log)
```

- **Semantics**: every subscriber gets a copy of every message
- **Use case**: event broadcasting (user signed up → notify multiple systems)
- **Examples**: Google Pub/Sub, Amazon SNS, Redis Pub/Sub (no persistence)

### Event Streams

An ordered, persistent, replayable log of events:

```
Producers ──→ [ Stream / Log ] ──→ Consumer Group A (position: 42)
                                ──→ Consumer Group B (position: 37)
```

- **Semantics**: consumers track their position (offset) in the stream. Can replay from any point
- **Key difference from queues**: messages aren't deleted after consumption. Multiple consumer groups read independently
- **Use case**: event sourcing, change data capture, real-time analytics
- **Examples**: Apache Kafka, Amazon Kinesis, Redis Streams

### Choosing Between Them

| Need | Pattern |
|------|---------|
| Distribute work across workers | Queue |
| Notify multiple independent consumers | Pub/Sub |
| Ordered, replayable event history | Stream |
| Fire-and-forget notifications | Pub/Sub |
| Exactly-once processing matters | Stream (with idempotent consumers) |

---

## Caching

Caching reduces latency and load by storing computed results closer to where they're needed.

### The Caching Layers

```
Client ──→ CDN ──→ Reverse Proxy ──→ App Cache ──→ DB Cache ──→ Database
              (edge)   (nginx)      (Redis/Memcached)  (buffer pool)
```

**CDN (Content Delivery Network)**:
- Caches static assets (images, JS, CSS) and sometimes API responses at edge locations close to users
- Reduces latency for geographically distributed users
- Cache invalidation via TTL, cache-busting URLs (append hash to filename), or purge APIs
- Examples: CloudFront, Cloudflare, Fastly

**Reverse Proxy Cache**:
- Sits in front of your application servers (Nginx, Varnish)
- Caches full HTTP responses based on URL, headers, and query params
- Good for responses that are identical across users (public pages, API listings)

**Application Cache**:
- In-process (local memory) or external (Redis, Memcached)
- You control exactly what gets cached, for how long, and when to invalidate
- Most flexible, most responsibility

**Database Cache (Buffer Pool)**:
- The database's internal cache — frequently accessed pages stay in memory
- You don't manage this directly, but you should be aware of it (a DB with enough RAM to hold the working set is dramatically faster)

### Cache Invalidation Strategies

```python
# Cache-Aside (Lazy Loading) — most common
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:                        # Cache miss
        user = db.query("SELECT * FROM users WHERE id = %s", user_id)
        cache.set(f"user:{user_id}", user, ttl=300)  # Cache for 5 min
    return user

# Write-Through — write to cache and DB together
def update_user(user_id, data):
    db.update("UPDATE users SET ... WHERE id = %s", user_id)
    cache.set(f"user:{user_id}", data, ttl=300)       # Update cache immediately

# Write-Behind (Write-Back) — write to cache, async write to DB
# Faster writes, but risk data loss if cache crashes before DB write
```

**The hard problem**: cache invalidation. When the underlying data changes, stale cache entries cause bugs. Strategies:
- **TTL (Time to Live)**: simplest. Accept staleness within the TTL window
- **Event-driven invalidation**: publish a "user updated" event, cache subscriber deletes the key
- **Read-through/Write-through**: the cache layer handles DB reads/writes, keeping itself consistent

---

## Rate Limiting

Protects your services from abuse and ensures fair resource usage.

### Token Bucket

Imagine a bucket that holds tokens. Each request consumes one token. Tokens are added at a fixed rate.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

Request arrives:
  - Tokens available? → Allow request, remove 1 token
  - No tokens? → Reject (429 Too Many Requests)

Allows short bursts (up to bucket capacity) while enforcing
an average rate (refill rate).
```

- **Pros**: allows bursts, simple to implement, predictable average rate
- **Used by**: AWS API Gateway, Stripe

### Leaky Bucket

Requests enter a queue (bucket) that drains at a constant rate. If the queue is full, new requests are dropped.

```
Requests → [ Queue (fixed size) ] → Processed at constant rate
              ↓ (if full)
           Rejected
```

- **Pros**: perfectly smooth output rate (no bursts)
- **Cons**: doesn't handle legitimate bursts well, adds latency (requests wait in queue)
- **Used by**: network traffic shaping, Nginx rate limiting

### Implementation Considerations

- **What to limit by**: API key, user ID, IP address, or a combination
- **Distributed rate limiting**: a single server's counter doesn't work behind a load balancer. Use Redis with atomic `INCR` and `EXPIRE`, or a dedicated rate limiting service
- **Response headers**: include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` so clients can back off gracefully

---

## Search: Inverted Indexes

Full-text search can't be done efficiently with SQL `LIKE '%query%'` — that's a full table scan.

### How Inverted Indexes Work

An inverted index maps every word to the list of documents containing it:

```
Documents:
  doc1: "the quick brown fox"
  doc2: "the lazy brown dog"
  doc3: "quick fox jumps"

Inverted Index:
  "the"   → [doc1, doc2]
  "quick" → [doc1, doc3]
  "brown" → [doc1, doc2]
  "fox"   → [doc1, doc3]
  "lazy"  → [doc2]
  "dog"   → [doc2]
  "jumps" → [doc3]

Search "quick fox":
  "quick" → [doc1, doc3]
  "fox"   → [doc1, doc3]
  Intersection → [doc1, doc3]
```

### Elasticsearch / OpenSearch

Built on top of Apache Lucene, these provide distributed, near-real-time search:

- **Documents** are stored as JSON, organized into **indexes** (like database tables)
- **Analyzers** process text before indexing: tokenize, lowercase, stem ("running" → "run"), remove stop words
- **Relevance scoring** (TF-IDF or BM25): ranks results by how well they match the query
- **Aggregations**: faceted search, histograms, statistics — analytics on top of search results

**When to use**: full-text search, log analysis, autocomplete, faceted product search.
**When NOT to use**: as a primary database. Elasticsearch is not ACID-compliant. Write to your primary DB, sync to Elasticsearch for search.

---

## Storage: Blob/Object Stores vs Block Storage

### Object/Blob Storage (S3, GCS, Azure Blob)

- Store files as opaque objects with metadata, accessed via HTTP (GET/PUT by key)
- Virtually unlimited capacity, pay per GB stored + operations
- **Durability**: designed for 99.999999999% (11 nines) durability
- **Access pattern**: whole-object read/write (no partial updates, no append)
- **Use cases**: images, videos, backups, static assets, data lake files (Parquet/CSV)

```
PUT /bucket/images/photo_001.jpg → stores the object
GET /bucket/images/photo_001.jpg → retrieves the whole object
```

### Block Storage (EBS, Persistent Disks)

- Virtual hard drives attached to a VM/instance
- Accessed as a raw block device — you format it with a filesystem (ext4, xfs)
- Low-latency, random read/write at the block level
- **Use cases**: database storage (PostgreSQL data directory), application file systems

### When to Use What

| Need | Choose |
|------|--------|
| Store user uploads (images, documents) | Object storage |
| Database data files | Block storage |
| Serve static website assets | Object storage + CDN |
| Application logs (archival) | Object storage |
| High-performance random I/O | Block storage |
| Share files across many services | Object storage |
| Mount a filesystem on a single VM | Block storage |

---

## Exercises

1. **Architecture Decision**: you're building a food delivery app with 3 engineers. Would you start with microservices or a monolith? What would trigger you to extract the first service? What would that first service be, and why?

2. **Caching Strategy**: an e-commerce product page shows product details, reviews, and "frequently bought together" recommendations. Design a caching strategy. Which caching layer handles what? What are the invalidation triggers for each?

3. **CQRS Evaluation**: a banking application has complex write rules (fraud detection, compliance checks, balance validation) but simple read needs (account balance, transaction history). Would CQRS help here? What would the write model and read model look like? What's the consistency implication?

4. **Messaging Choice**: you need to (a) send welcome emails after signup, (b) sync user data to 4 downstream services, (c) process video uploads. For each, choose queue, pub/sub, or stream, and justify your choice.

5. **Search Design**: your app has 10 million product listings and users search by name, category, price range, and brand. Design the search infrastructure. What data goes in the inverted index? How do you keep it in sync with the product database?

---

## Recommended Resources

- **Designing Data-Intensive Applications** by Martin Kleppmann — chapters on replication, partitioning, and derived data directly cover CQRS, event sourcing, and stream processing
- **Building Microservices** (2nd Edition) by Sam Newman — the pragmatic guide to when/how to decompose services, including the modular monolith approach
- **System Design Interview** by Alex Xu — walks through common design problems (rate limiter, chat system, news feed) with concrete architecture decisions
- **Martin Fowler's blog** (martinfowler.com) — definitive articles on CQRS, event sourcing, microservices, and the "Monolith First" pattern
- **The Amazon Builders' Library** — "Caching challenges and strategies," "Avoiding fallback in distributed systems," and other battle-tested patterns
- **Elasticsearch: The Definitive Guide** — free online book covering inverted indexes, analysis, and search relevance in depth
- **Kafka: The Definitive Guide** — covers streams vs queues, consumer groups, and exactly-once semantics with practical examples
