# Human and Software Systems

Software doesn't exist in a vacuum. It's built by teams, used by people, and evolves over years. The most elegant architecture is worthless if requirements were misunderstood, the API breaks every client on upgrade, or the team can't maintain the codebase. This lesson covers the human side of software engineering: gathering requirements, making tradeoffs, documenting decisions, designing APIs that last, and managing the inevitable accumulation of technical debt.

---

## Requirements: Functional vs Non-Functional

Before writing code, you need to understand *what* you're building and *how well* it needs to work. These are two fundamentally different kinds of requirements.

### Functional Requirements

**What the system does** — the features and behaviors:

- "Users can create an account with email and password"
- "The system sends a confirmation email within 30 seconds of signup"
- "Admins can export all user data as CSV"
- "The shopping cart persists across sessions"

Functional requirements are what stakeholders talk about naturally. They describe observable behavior from the user's perspective.

### Non-Functional Requirements (Quality Attributes)

**How well the system does it** — the constraints on behavior:

| Category | Examples |
|---|---|
| **Performance** | API responds in < 200ms at p95; supports 10,000 concurrent users |
| **Availability** | 99.9% uptime (43 min/month downtime budget) |
| **Scalability** | Handle 10x traffic growth without architecture changes |
| **Security** | All data encrypted at rest; SOC 2 compliant |
| **Maintainability** | New team member productive within 2 weeks |
| **Accessibility** | WCAG 2.1 AA compliance |
| **Compliance** | GDPR: user data deletion within 30 days of request |

### Why Non-Functional Requirements Matter More Than You Think

Most system failures aren't because a feature is missing — they're because the system is too slow, can't scale, is impossible to maintain, or has a security hole. Non-functional requirements:

- **Drive architecture**: "supports 10M users" implies very different architecture than "supports 100 users"
- **Are hard to add later**: making a system 10x faster after it's built is much harder than designing for it upfront
- **Are often implicit**: stakeholders assume the system will be "fast" and "reliable" without stating numbers. **Make them explicit.** Ask: "How fast? How reliable? What's the cost of 1 hour of downtime?"

```
# Bad requirement
"The system should be fast."

# Good requirement
"The product search API returns results in < 150ms at p95 under 
5,000 concurrent users, with a linear scaling path to 50,000."
```

---

## Tradeoffs: Build vs Buy, Simplicity vs Flexibility

Software engineering is the art of making tradeoffs under uncertainty. Every decision has costs that aren't immediately visible.

### Build vs Buy

| Factor | Build | Buy/Use Existing |
|---|---|---|
| **Customization** | Exactly what you need | Must work within vendor's model |
| **Time to market** | Weeks/months to build | Days to integrate |
| **Maintenance** | You own all bugs and upgrades | Vendor handles infrastructure |
| **Cost** | Engineering time (expensive) | Licensing fees (predictable) |
| **Risk** | You might build the wrong thing | Vendor might sunset the product |
| **Knowledge** | Deep understanding of the system | Black box behavior |

**Decision framework**:

- **Buy** when: it's not your core competency, good solutions exist, time-to-market matters, you lack domain expertise
- **Build** when: it's core to your competitive advantage, existing solutions don't fit, you need deep control, vendor lock-in is unacceptable

**Real examples**:
- **Buy**: authentication (Auth0/Clerk), payment processing (Stripe), email sending (SendGrid), monitoring (Datadog)
- **Build**: your core product logic, proprietary algorithms, domain-specific workflows that no vendor supports

### Simplicity vs Flexibility

The desire for flexibility is one of the biggest sources of over-engineering:

```python
# Over-engineered: "we might need different notification channels someday"
class NotificationStrategyFactory:
    def create_strategy(self, channel_type: str) -> NotificationStrategy:
        registry = self._get_strategy_registry()
        return registry.get(channel_type, DefaultStrategy())()

# Simple: you only send emails right now
def send_notification(user_email: str, message: str):
    send_email(user_email, message)
```

**YAGNI** (You Aren't Gonna Need It): don't build abstractions for requirements you don't have yet. When the need actually arises, you'll understand the problem better and build a more appropriate solution.

**Exception**: when the cost of change is truly high (database schema, public API contract, data format), invest more upfront in getting it right.

---

## Documentation: ADRs and Runbooks

Good documentation is a force multiplier. Bad documentation (or none at all) creates single points of failure in people's heads.

### Architecture Decision Records (ADRs)

An ADR captures **why** a significant technical decision was made — not just what was decided, but what alternatives were considered and what trade-offs were accepted.

```markdown
# ADR-007: Use PostgreSQL for primary data store

## Status
Accepted (2025-01-15)

## Context
We need a primary database for our SaaS application. Expected load 
is ~1000 QPS with mostly read-heavy workloads. Team has experience 
with both PostgreSQL and MySQL. We also considered MongoDB.

## Decision
Use PostgreSQL 16 on AWS RDS.

## Alternatives Considered
- **MySQL**: viable, but Postgres has better JSON support, window 
  functions, and partial indexes — all useful for our analytics features
- **MongoDB**: flexible schema is appealing for rapid prototyping, but 
  our data is relational and we'd lose transaction guarantees we need

## Consequences
- (+) Strong ecosystem, excellent documentation, mature tooling
- (+) JSONB columns give us schema flexibility where needed
- (-) Vertical scaling has limits — may need read replicas by year 2
- (-) Team must learn Postgres-specific features (VACUUM, pg_stat, etc.)
```

**Why ADRs matter**:
- **Future you** (or a new team member) will wonder "why did we choose X?" The ADR answers this without tracking down the person who made the decision
- **Prevent re-litigation**: without a record, the same decision gets debated every 6 months
- **Capture context**: decisions that seem wrong now often made perfect sense given the constraints at the time

### Runbooks

A runbook is a step-by-step guide for handling operational tasks or incidents. It should be written so that an on-call engineer at 3 AM — tired, stressed, and unfamiliar with this subsystem — can follow it.

```markdown
# Runbook: Database connection pool exhaustion

## Symptoms
- Application logs show "connection pool exhausted" errors
- Monitoring shows active_db_connections at or near max (default: 100)
- Elevated 5xx error rate on API responses

## Diagnosis
1. Check current connections:
   SELECT count(*), state FROM pg_stat_activity GROUP BY state;
2. Find long-running queries:
   SELECT pid, now() - query_start AS duration, query
   FROM pg_stat_activity WHERE state = 'active'
   ORDER BY duration DESC LIMIT 10;
3. Check for connection leaks in application logs:
   grep "connection not returned" /var/log/app/*.log

## Mitigation
1. Kill long-running queries (if safe):
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE duration > interval '5 minutes' AND state = 'active';
2. Restart the application to reset the connection pool
3. If recurring, increase pool size in config (max_connections in 
   pgbouncer.ini) — but this is a band-aid

## Root Cause Follow-up
- Check recent deployments for missing connection.close() calls
- Review slow query log for queries that started hanging
- Verify pgbouncer configuration matches application pool settings
```

**Key qualities of a good runbook**: specific commands (not "check the database"), copy-pasteable, no assumed knowledge, includes escalation paths.

---

## API Design: Versioning, Compatibility, and Deprecation

APIs are contracts with your consumers. Breaking that contract breaks their software, their trust, and your relationship.

### Design for Longevity

```
# Good: clear, consistent, predictable
GET    /api/v1/orders              # list orders
GET    /api/v1/orders/123          # get order by ID
POST   /api/v1/orders              # create order
PATCH  /api/v1/orders/123          # partial update
DELETE /api/v1/orders/123          # delete order

# Bad: inconsistent naming, verbs in URLs
GET    /api/getOrders
POST   /api/createNewOrder
PUT    /api/order/update/123
GET    /api/v1/order_list?action=delete&id=123
```

**Naming conventions**: use plural nouns for collections (`/orders`, not `/order`), consistent casing (kebab-case or snake_case, pick one), logical nesting (`/users/123/orders` for a user's orders).

### Versioning Strategies

**URL versioning** (`/api/v1/orders`):
- Simple, explicit, easy to route
- Consumers always know which version they're using
- Downside: URL changes when version bumps, which breaks bookmarks and caches

**Header versioning** (`Accept: application/vnd.myapi.v2+json`):
- Clean URLs, version is metadata
- Downside: harder to test (curl needs extra headers), less visible

**Query parameter** (`/api/orders?version=2`):
- Simple to implement
- Downside: mixing concerns, easy to forget

**Practical recommendation**: URL versioning for external/public APIs (simplicity wins). Header versioning for internal APIs if you prefer clean URLs.

### Backward Compatibility

The golden rule: **additive changes are safe, removals and renames are breaking**.

```json
// v1 response
{ "id": 123, "name": "Widget", "price": 9.99 }

// v1.1 response — SAFE (additive)
{ "id": 123, "name": "Widget", "price": 9.99, "currency": "USD" }

// v2 response — BREAKING (field renamed)
{ "id": 123, "title": "Widget", "price_cents": 999 }
```

**Safe (non-breaking) changes**:
- Adding new optional fields to responses
- Adding new optional query parameters
- Adding new endpoints
- Adding new enum values (if clients handle unknowns gracefully)

**Breaking changes**:
- Removing or renaming fields
- Changing field types (`string` → `number`)
- Changing required/optional status of request fields
- Changing error response formats
- Changing authentication mechanisms

### Deprecation

When you need to make breaking changes, deprecate the old version before removing it:

1. **Announce**: document the deprecation with a timeline (e.g., "v1 deprecated, removed on 2025-07-01")
2. **Signal**: return deprecation headers so clients are aware

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jul 2025 00:00:00 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

3. **Monitor**: track how many clients still use the deprecated version
4. **Migrate**: help major consumers migrate (documentation, migration guides)
5. **Remove**: only after the sunset date and when usage is negligible

---

## Code Review Culture

Code review isn't about gatekeeping — it's about shared understanding, knowledge transfer, and catching issues that automated tools miss.

### What Good Code Review Looks Like

**Reviewer responsibilities**:
- **Focus on correctness and design**, not style (use formatters and linters for style)
- **Ask questions, don't demand**: "What happens if this list is empty?" is better than "This is wrong"
- **Approve when it's good enough**, not perfect. Don't block on nitpicks
- **Review promptly**: a PR that sits for 3 days blocks the author and invites merge conflicts

**Author responsibilities**:
- **Keep PRs small**: 50–200 lines is reviewable. 2,000 lines gets rubber-stamped
- **Write a good description**: what changed, why, and how to test it
- **Self-review first**: read your own diff before requesting review
- **Separate refactoring from features**: a PR that refactors *and* adds a feature is hard to review

### What to Look For

```
Priority 1 (must fix):
  - Bugs: logic errors, race conditions, missing error handling
  - Security: injection, auth bypass, secrets in code
  - Data issues: missing validation, potential data corruption

Priority 2 (should fix):
  - Design: poor abstractions, wrong layer, missing encapsulation
  - Performance: N+1 queries, unbounded loops, missing indexes
  - Reliability: missing retries, no timeout, no circuit breaker

Priority 3 (nice to fix):
  - Readability: confusing names, missing context comments
  - Testing: missing edge cases, brittle assertions
  - Consistency: deviating from team patterns without reason
```

---

## Technical Debt Management

Technical debt is the gap between how your code *is* and how it *should be*. Some debt is intentional (shipping fast), some is accidental (we didn't know better), and some is environmental (requirements changed).

### Types of Technical Debt

- **Deliberate**: "We know this isn't ideal, but we need to ship by Friday. We'll refactor next sprint." (Often fine — if you actually pay it back)
- **Accidental**: "We didn't realize this design wouldn't scale." (Unavoidable — but learn from it)
- **Bit rot**: "This was fine when we wrote it, but the rest of the system evolved and now it's awkward." (Natural, constant)
- **Environmental**: "The library we depend on is deprecated." (External forces)

### Managing Debt Practically

**Track it**: maintain a lightweight debt registry — a list of known debt with severity and estimated cost to fix.

```markdown
| ID | Description | Severity | Effort | Status |
|----|-------------|----------|--------|--------|
| D-1 | Auth service uses deprecated JWT lib | High | 3 days | Planned |
| D-2 | Order service has no retry logic | Medium | 1 day | Backlog |
| D-3 | Frontend build takes 8 minutes | Low | 2 days | Backlog |
```

**Pay it down continuously**: don't save all debt work for a mythical "tech debt sprint." Instead:

- **Boy Scout Rule**: leave the code a little better than you found it. If you're modifying a file with debt, fix some while you're there
- **20% allocation**: dedicate ~20% of sprint capacity to debt reduction
- **Tie debt to features**: "We need to refactor the payment module *because* adding the new payment method on top of the current design would take 3x longer"
- **Know when to not pay it**: debt in code that's stable and rarely changed is low priority. Don't refactor working code just because it's "ugly"

### When Technical Debt Becomes Critical

- **Deployment frequency decreasing**: a sign that the codebase is too fragile to change safely
- **Onboarding time increasing**: new engineers take weeks to become productive
- **Bug rate increasing**: each fix introduces new bugs because the code is hard to reason about
- **Feature velocity declining**: simple features take disproportionately long because they require navigating layers of workarounds

These are signals that debt has compounded to the point where it's actively harming the business. At this point, make the business case: "We can keep adding features at 30% of our historical velocity, or we can invest 6 weeks in this refactor and return to full velocity."

---

## Exercises

1. **Requirements gathering**: pick a feature you've recently built or are planning to build. Write 5 functional requirements and 5 non-functional requirements. For each non-functional requirement, include a specific, measurable target (not "fast" but "< 200ms at p95").

2. **ADR writing**: write an ADR for a recent technical decision your team made (language choice, framework selection, database migration, architecture change). Include context, alternatives considered, and consequences. Share it with a colleague and see if they can understand the reasoning without additional explanation.

3. **API review**: review the API of a service you maintain. Identify: (a) any endpoints that would be breaking changes if removed, (b) any inconsistencies in naming or patterns, (c) fields in responses that should have been optional but are required. Draft a deprecation plan for one problematic endpoint.

4. **Technical debt audit**: pick a codebase you work on. List 5 items of technical debt, categorize each (deliberate, accidental, bit rot, environmental), estimate the effort to fix, and rank them by impact-to-effort ratio. Which one would you tackle first?

5. **Code review calibration**: take a PR you reviewed recently (or an open-source PR). Re-review it, categorizing each comment as Priority 1, 2, or 3 (using the framework above). Were you blocking on the right things? Were you spending review time on things a linter could catch?

---

## Recommended Resources

- **"A Philosophy of Software Design" by John Ousterhout** — concise, opinionated guide to software design that emphasizes managing complexity. Short and re-readable.
- **"Documenting Software Architectures" by Clements et al.** — comprehensive guide to architecture documentation, including ADR templates and viewpoint frameworks.
- **ADR GitHub repo ([github.com/joelparkerhenderson/architecture-decision-record](https://github.com/joelparkerhenderson/architecture-decision-record))** — collection of ADR templates and real-world examples.
- **"Refactoring" by Martin Fowler** — the classic guide to improving existing code systematically, with a catalog of refactoring patterns.
- **Google's API Design Guide ([cloud.google.com/apis/design](https://cloud.google.com/apis/design))** — practical conventions for designing clean, consistent APIs. Applicable far beyond Google Cloud.
- **"Kill It with Fire" by Marianne Bellotti** — practical strategies for modernizing legacy systems and managing technical debt at scale.
