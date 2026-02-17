# Databases

Almost every application stores and retrieves data, and the database is where the hardest trade-offs live. Choosing the right data model, writing efficient queries, understanding transaction guarantees, and planning for scale are skills that separate engineers who build reliable systems from those who build fragile ones. Database decisions are also among the most expensive to change later — getting them reasonably right early saves enormous pain.

---

## Data Modeling

Data modeling is the process of deciding how to structure your data. Get this wrong, and every query you write will fight you.

### Entities, Relations, and Keys

- **Entities** are the "things" in your domain: users, orders, products, messages
- **Relations** describe how entities connect: a user *places* orders, an order *contains* products
- **Primary key**: uniquely identifies a row (e.g., `user_id`). Should be immutable and compact
- **Foreign key**: references a primary key in another table, enforcing referential integrity
- **Natural vs surrogate keys**: natural keys use real data (`email`), surrogate keys are generated (`UUID`, auto-increment). Surrogate keys are almost always preferred — natural keys change

### Normalization vs Denormalization

**Normalization** (splitting data into related tables) reduces duplication and anomalies:

```sql
-- Normalized: separate tables
CREATE TABLE users (
    user_id   SERIAL PRIMARY KEY,
    name      TEXT NOT NULL,
    email     TEXT UNIQUE NOT NULL
);

CREATE TABLE orders (
    order_id  SERIAL PRIMARY KEY,
    user_id   INT REFERENCES users(user_id),
    total     DECIMAL(10, 2),
    created   TIMESTAMP DEFAULT now()
);
```

**Denormalization** (duplicating data) optimizes for read performance:

```sql
-- Denormalized: user name embedded in orders
CREATE TABLE orders (
    order_id   SERIAL PRIMARY KEY,
    user_id    INT,
    user_name  TEXT,      -- duplicated from users table
    user_email TEXT,      -- duplicated from users table
    total      DECIMAL(10, 2),
    created    TIMESTAMP DEFAULT now()
);
```

**Trade-off**: normalization = data integrity + write simplicity; denormalization = read performance + query simplicity but update complexity (must update duplicates everywhere).

### OLTP vs OLAP

| Characteristic | OLTP | OLAP |
|---------------|------|------|
| Purpose | Run the business (transactions) | Analyze the business (reporting) |
| Queries | Short, frequent, by primary key | Long, complex, full-table scans |
| Schema | Normalized (3NF) | Denormalized (star/snowflake schema) |
| Examples | Postgres, MySQL | BigQuery, Redshift, ClickHouse |
| Rows touched | Tens | Millions |

Most applications start with OLTP. When analytics queries start slowing down your production database, it's time to add an OLAP system.

### Constraints, Indexes, and Cardinality

- **Constraints** enforce data rules at the database level: `NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`
- **Indexes** speed up reads at the cost of slower writes and more storage
- **Cardinality**: the number of distinct values in a column. High cardinality (e.g., `user_id`) = good index candidate. Low cardinality (e.g., `status` with 3 values) = usually poor index candidate

---

## SQL Fundamentals

SQL is the language of relational databases. Mastering it means mastering the most battle-tested data tool in existence.

### Joins

```sql
-- INNER JOIN: only matching rows from both tables
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id;

-- LEFT JOIN: all users, even those without orders (NULLs for missing)
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id;

-- Self-join: employees and their managers
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

### Grouping and Aggregation

```sql
-- Total revenue per user, only users with > $1000
SELECT u.name, SUM(o.total) AS revenue
FROM users u
JOIN orders o ON u.user_id = o.user_id
GROUP BY u.name
HAVING SUM(o.total) > 1000
ORDER BY revenue DESC;
```

Key distinction: `WHERE` filters rows *before* grouping; `HAVING` filters groups *after* aggregation.

### Window Functions

Window functions compute values across a set of rows related to the current row — without collapsing them into groups:

```sql
-- Rank users by total spending
SELECT
    user_id,
    total,
    SUM(total) OVER (PARTITION BY user_id) AS user_total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created DESC) AS rn,
    RANK() OVER (ORDER BY total DESC) AS spending_rank
FROM orders;

-- Running total (cumulative sum)
SELECT
    created::date AS day,
    total,
    SUM(total) OVER (ORDER BY created) AS running_total
FROM orders;
```

Window functions are one of the most powerful and underused features of SQL. They eliminate the need for many self-joins and subqueries.

### Transactions

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;
-- If anything fails, ROLLBACK undoes both operations
```

Transactions ensure a group of operations either all succeed or all fail — no partial updates.

### Query Plans

The **query planner** decides how to execute your SQL. Learning to read its output is essential:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42 AND status = 'shipped';
```

Key things to look for:

- **Seq Scan** (sequential scan): reads every row in the table. Fine for small tables, terrible for large ones
- **Index Scan / Index Only Scan**: uses an index to jump directly to matching rows. This is what you want
- **Bitmap Index Scan**: combines results from multiple indexes
- **Nested Loop / Hash Join / Merge Join**: different join strategies with different performance characteristics
- **Rows** (estimated vs actual): if the estimate is wildly off, statistics are stale — run `ANALYZE`
- **Selectivity**: how much of the table a condition filters. `status = 'active'` on a column where 90% of rows are active has low selectivity — an index won't help much

---

## Indexing

Indexes are the primary tool for making queries fast. But they're not free.

### B-Tree / B+Tree

The default index type in most databases (Postgres, MySQL, SQLite):

- Keeps data sorted in a balanced tree structure
- Supports equality (`=`) and range queries (`<`, `>`, `BETWEEN`, `ORDER BY`)
- **B+Tree** (most common): data stored only in leaf nodes, which are linked for efficient range scans
- Lookup time: **O(log n)** — a table with 1 billion rows needs ~30 comparisons

```sql
-- Creates a B-Tree index by default
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

### Composite Indexes

An index on multiple columns. Column **order matters critically**:

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
```

This index supports:
- `WHERE user_id = 42` — yes (leftmost prefix)
- `WHERE user_id = 42 AND status = 'shipped'` — yes (full match)
- `WHERE status = 'shipped'` — **no** (can't skip the leftmost column)

Think of it like a phone book sorted by last name, then first name. You can look up by last name, or by last name + first name, but not by first name alone.

### Covering Indexes

An index that includes all columns a query needs, so the database never touches the main table:

```sql
-- Query: SELECT status, total FROM orders WHERE user_id = 42
CREATE INDEX idx_orders_covering ON orders (user_id) INCLUDE (status, total);
```

This is an **index-only scan** — much faster because it avoids the random I/O of looking up the full row.

### Hash Indexes

- Support only equality lookups (`=`), not ranges
- O(1) average lookup time
- Available in Postgres (but B-Tree is almost always preferred due to flexibility)
- Used internally by hash joins in query execution

### Common Indexing Pitfalls

1. **Wrong column order in composite indexes**: `(status, user_id)` doesn't help `WHERE user_id = ?` queries. Always put the highest-selectivity, most-queried column first

2. **Indexing low-selectivity columns**: an index on `gender` (2 values) or `is_active` (boolean) rarely helps — the database will just scan the table anyway. Exception: partial indexes

3. **Too many indexes**: every index slows down writes (`INSERT`, `UPDATE`, `DELETE`) and consumes storage. Index what you query, not everything

4. **Functions defeating indexes**: `WHERE LOWER(email) = 'foo@bar.com'` cannot use an index on `email`. Use an expression index instead:

```sql
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
```

5. **Not analyzing**: the query planner relies on statistics. After bulk loads or major changes, run `ANALYZE` to update them

---

## Transactions and Isolation

### ACID Properties

- **Atomicity**: a transaction is all-or-nothing
- **Consistency**: the database moves from one valid state to another (constraints are enforced)
- **Isolation**: concurrent transactions don't interfere with each other (to varying degrees)
- **Durability**: once committed, data survives crashes

### Write-Ahead Logging (WAL)

How databases achieve durability without writing to disk on every operation:

1. Before modifying data, write the change to a sequential **log file** (the WAL)
2. Acknowledge the commit once the WAL entry is flushed to disk
3. Apply the changes to the actual data files later (in the background)
4. On crash recovery, replay the WAL to restore any committed but unapplied changes

Sequential writes to the WAL are much faster than random writes to data files. This is the key insight.

### Isolation Levels

Isolation levels control what concurrent transactions can see:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|-------------------|-------------|
| Read Uncommitted | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Possible |
| **Serializable** | Prevented | Prevented | Prevented |

**Anomalies explained**:

- **Dirty read**: reading data from an uncommitted transaction (it might roll back)
- **Non-repeatable read**: reading the same row twice in a transaction and getting different values (someone else committed a change in between)
- **Phantom read**: running the same query twice and getting different *rows* (someone inserted/deleted matching rows)

```sql
-- Set isolation level for a transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT balance FROM accounts WHERE id = 1;  -- reads 500
    -- Another transaction commits: UPDATE accounts SET balance = 300 WHERE id = 1
    SELECT balance FROM accounts WHERE id = 1;  -- still reads 500 (snapshot)
COMMIT;
```

**Postgres default**: Read Committed. **MySQL/InnoDB default**: Repeatable Read.

### MVCC (Multi-Version Concurrency Control)

Instead of locking rows, MVCC keeps multiple versions of each row:

- Each transaction sees a **snapshot** of the database as of its start time
- Readers never block writers; writers never block readers
- Old versions are cleaned up by a background process (Postgres: `VACUUM`, MySQL: purge thread)

MVCC is how Postgres and MySQL achieve high concurrency without heavy locking. The trade-off is storage overhead from old row versions and the need for periodic cleanup.

---

## NoSQL Families

Relational databases are the default, but sometimes a different data model fits better.

### Key-Value Stores

- **Model**: simple `key → value` mapping
- **Examples**: Redis, DynamoDB, etcd
- **Best for**: caching, session storage, configuration, counters
- **Limitation**: no queries by value — you must know the key

### Document Stores

- **Model**: `key → JSON/BSON document` with nested structure
- **Examples**: MongoDB, CouchDB, Firestore
- **Best for**: content management, user profiles, event data — anything with variable schema
- **Limitation**: joins are expensive or impossible; relationships must be embedded or handled by the application

### Wide-Column Stores

- **Model**: rows with dynamic columns, grouped into column families
- **Examples**: Cassandra, HBase, ScyllaDB
- **Best for**: time-series, IoT, high write throughput at massive scale
- **Limitation**: limited query flexibility — must design tables around access patterns

### Graph Databases

- **Model**: nodes and edges with properties
- **Examples**: Neo4j, Amazon Neptune, Dgraph
- **Best for**: social networks, fraud detection, recommendation engines — anything where relationships are the primary query target
- **Limitation**: less mature tooling; not suited for bulk analytics

### Time-Series Databases

- **Model**: optimized for timestamped append-only data
- **Examples**: InfluxDB, TimescaleDB (Postgres extension), Prometheus
- **Best for**: metrics, monitoring, IoT sensor data
- **Limitation**: poor fit for non-temporal queries

### NoSQL Trade-offs

| | Relational | NoSQL |
|---|---|---|
| Schema | Rigid, enforced | Flexible, schema-on-read |
| Consistency | Strong (ACID) | Often eventual |
| Joins | Native, efficient | Application-level or absent |
| Scaling | Vertical (primarily) | Horizontal (designed for it) |
| Query language | SQL (standardized) | Varies per system |

**Rule of thumb**: start with a relational database. Move to NoSQL when you have a specific need it can't meet (e.g., massive write throughput, graph traversals, sub-millisecond key lookups at scale).

---

## Replication and Sharding

Single-server databases eventually hit limits: either too much data, too many queries, or insufficient fault tolerance.

### Replication

**Primary/replica** (leader/follower) replication:

```
Writes → [Primary] ──replication──→ [Replica 1] ← Reads
                    ──replication──→ [Replica 2] ← Reads
```

- **Synchronous replication**: primary waits for replica to confirm. Guarantees consistency but adds write latency
- **Asynchronous replication**: primary doesn't wait. Lower latency but replicas may lag (eventual consistency)
- **Semi-synchronous**: primary waits for at least one replica (balance between the two)

**Failover**: when the primary dies, a replica is promoted. This can be:
- **Manual**: operator decides (safest but slowest)
- **Automatic**: consensus-based election (e.g., Patroni for Postgres, MySQL Group Replication)
- Risk: if the old primary comes back, it may have conflicting writes (split-brain)

### Sharding (Partitioning)

Splitting data across multiple databases, each holding a subset:

**Hash partitioning**: `shard = hash(key) % num_shards`
- Even distribution
- No range queries across shards
- Resharding when adding nodes requires moving data

**Range partitioning**: `shard_1: A-M, shard_2: N-Z`
- Supports range scans within a shard
- Risk of **hot partitions** (e.g., all new users this month start with the same letter)

**Consistent hashing**: minimizes data movement when adding/removing nodes
- Used by DynamoDB, Cassandra, and many distributed caches
- Each node owns a range on a hash ring; adding a node only moves data from its neighbors

### Sharding Challenges

- **Cross-shard queries**: `JOIN` across shards is expensive — requires scatter-gather
- **Hot partitions**: one shard gets disproportionate traffic (e.g., a viral user's data). Solutions: further splitting, random suffixing of hot keys
- **Rebalancing**: adding or removing shards requires data migration, which must happen without downtime
- **Transactions**: distributed transactions across shards (2PC) are slow and complex. Most systems avoid them by designing shard keys so related data lives on the same shard

```sql
-- Example: shard by user_id so all of a user's data is co-located
-- This means user queries never cross shards
-- But queries like "all orders today" require querying every shard
```

**Practical advice**: avoid sharding as long as possible. A single Postgres instance with proper indexing, connection pooling, and read replicas handles far more than most people expect (often millions of rows). Shard only when you've exhausted vertical scaling, read replicas, and caching.

---

## Exercises

1. **Normalization practice**: take a denormalized spreadsheet-style table (e.g., `orders` with customer name, email, product name, and category repeated per row). Normalize it into 3NF with proper foreign keys. Then write the `JOIN` query to reconstruct the original view.

2. **Query plan analysis**: create a table with 1 million rows. Write a query with a `WHERE` clause and run `EXPLAIN ANALYZE` with and without an index. Compare the execution time, the plan (Seq Scan vs Index Scan), and the estimated vs actual row counts.

3. **Window functions**: given an `orders` table, write queries to:
   - Rank each customer's orders by amount
   - Calculate a running total per customer
   - Find each customer's most recent order without a subquery

4. **Isolation level experiment**: open two `psql` sessions. In one, start a `REPEATABLE READ` transaction and read a row. In the other, update that row and commit. Read the row again in the first session. What do you see? Repeat with `READ COMMITTED`. What changes?

5. **Sharding design**: you're building a multi-tenant SaaS app with 10,000 tenants. Some tenants have 100 users, some have 100,000. Design a sharding strategy. What's your shard key? How do you handle cross-tenant analytics? What happens when one tenant goes viral?

6. **NoSQL evaluation**: for each scenario, pick the best database type and justify:
   - Real-time multiplayer game leaderboard
   - Social network friend-of-friend recommendations
   - IoT sensor data from 100,000 devices
   - E-commerce product catalog with variable attributes

---

## Recommended Resources

- **"Designing Data-Intensive Applications" by Martin Kleppmann** — the single best book on databases, replication, partitioning, and distributed data. Required reading.
- **"Use The Index, Luke" ([use-the-index-luke.com](https://use-the-index-luke.com))** — free online book focused entirely on SQL indexing. Incredibly practical.
- **PostgreSQL documentation** — the best database documentation in the industry. The chapters on indexing, query planning, and MVCC are excellent.
- **CMU Database Group (YouTube)** — Andy Pavlo's database systems lectures. Academic rigor with practical relevance.
- **Percona Blog** — deep dives into MySQL and Postgres internals, performance tuning, and operations.
- **"The Art of PostgreSQL" by Dimitri Fontaine** — advanced SQL techniques, window functions, and practical patterns.
