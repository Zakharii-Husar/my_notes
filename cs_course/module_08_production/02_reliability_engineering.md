# Reliability Engineering

Building software is half the job. Keeping it running, detecting problems before users do, and recovering gracefully when things break — that's reliability engineering. This discipline emerged from the reality that distributed systems fail constantly: disks die, networks partition, deployments introduce bugs, and traffic spikes arrive without warning. Reliability engineering gives you the tools and frameworks to turn chaos into manageable risk.

---

## SLO / SLI / SLA

These three concepts form the foundation of reliability thinking. They shift the conversation from "is the system up?" to "is the system meeting user expectations?"

### Service Level Indicators (SLIs)

An SLI is a **quantitative measurement** of some aspect of service health. It's what you actually measure.

- **Availability**: percentage of successful requests (`successes / total`)
- **Latency**: percentage of requests faster than a threshold (e.g., p99 < 300ms)
- **Error rate**: percentage of requests returning errors
- **Throughput**: requests per second served successfully

```
SLI (availability) = (total_requests - error_requests) / total_requests
SLI (latency)      = requests_under_300ms / total_requests
```

### Service Level Objectives (SLOs)

An SLO is a **target value** for an SLI. It's what you commit to internally.

- "99.9% of requests will succeed over a 30-day window"
- "95% of API requests will complete in under 200ms"

**Why not 100%?** Because 100% reliability is infinitely expensive and prevents you from shipping changes. An SLO of 99.9% means you accept ~43 minutes of downtime per month — and that's the budget you spend on deploying new features.

### Service Level Agreements (SLAs)

An SLA is a **contractual commitment** — an SLO with consequences (refunds, credits, penalties). SLAs should always be less aggressive than internal SLOs so you have a buffer.

```
SLA (external promise):  99.9% availability
SLO (internal target):   99.95% availability  ← tighter, gives you margin
SLI (actual measurement): 99.97% this month   ← measured reality
```

---

## Error Budgets

An error budget is the inverse of your SLO: the amount of unreliability you're *allowed*.

```
SLO = 99.9% availability over 30 days
Error budget = 0.1% = ~43 minutes of downtime per month
```

**How to use error budgets**:

- **Budget remaining**: ship features, deploy faster, take risks
- **Budget exhausted**: freeze deployments, focus on reliability work, slow down
- **Budget consistently unspent**: your SLO might be too loose, or you're being too cautious

Error budgets turn the reliability-vs-velocity debate into a data-driven conversation. Instead of arguing about whether to deploy on Friday, you check the error budget.

| Error Budget Status | Action |
|---|---|
| Plenty remaining | Ship features, experiment |
| Getting low | Increase testing, slower rollouts |
| Exhausted | Deployment freeze, reliability focus |

---

## Incident Response

When things break (and they will), the difference between a minor blip and a catastrophe is how you respond.

### Paging and Alerting

- **Page only on user-facing impact**: if no user is affected, it's not worth waking someone up. Alert, don't page.
- **Route to the right team**: alerts about database latency go to the database team, not the frontend team
- **Avoid alert fatigue**: if an alert fires 10 times a day and is always ignored, it's worse than no alert — it trains people to ignore all alerts

```yaml
# Good alert: specific, actionable, tied to user impact
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
for: 5m
labels:
  severity: page
annotations:
  summary: "Error rate above 1% for 5 minutes — users are affected"
```

### Triage

When an incident starts:

1. **Assess impact**: how many users? Which features? Is it getting worse?
2. **Mitigate first, debug later**: roll back the last deploy, toggle a feature flag, scale up, redirect traffic. Stop the bleeding before finding the root cause
3. **Communicate**: update the status page, notify stakeholders, set expectations
4. **Assign roles**: incident commander (coordinates), technical lead (debugs), communicator (updates stakeholders)

### Postmortems

After every significant incident, write a blameless postmortem:

- **What happened**: timeline of events with timestamps
- **Impact**: duration, affected users, financial impact
- **Root cause**: the underlying issue (not "human error" — that's never a root cause)
- **Contributing factors**: what made detection or recovery slow
- **Action items**: concrete, assigned, time-bound improvements
- **What went well**: also document what worked — good monitoring, fast rollback, effective communication

**Blameless** is critical. If people fear blame, they hide information, and you learn nothing. The goal is to improve the *system*, not punish individuals.

---

## Observability

Observability is the ability to understand what's happening inside your system by examining its outputs. The three pillars are logs, metrics, and traces — but the real power comes from correlating them.

### Logs

Structured events from your application:

```json
{
  "timestamp": "2025-03-15T14:30:22Z",
  "level": "error",
  "service": "payment-service",
  "trace_id": "abc123",
  "user_id": "user_456",
  "message": "Payment processing failed",
  "error": "gateway_timeout",
  "duration_ms": 30012
}
```

**Best practices**:
- Use **structured logging** (JSON) — not free-text messages that require regex to parse
- Include **correlation IDs** (`trace_id`, `request_id`) in every log line
- Log at the right level: `ERROR` for failures, `WARN` for degraded behavior, `INFO` for significant events, `DEBUG` for development only

### Metrics

Numeric measurements aggregated over time:

```
# Prometheus-style metrics
http_requests_total{method="POST", endpoint="/api/orders", status="200"} 15234
http_request_duration_seconds{quantile="0.99"} 0.45
active_db_connections 23
```

Metrics are cheap to store (they aggregate), fast to query, and ideal for dashboards and alerts. Logs tell you *what happened*; metrics tell you *how much* and *how often*.

### Traces

A trace follows a single request as it flows through multiple services:

```
[Trace: abc123]
├─ API Gateway (12ms)
│  └─ Auth Service (3ms)
├─ Order Service (45ms)
│  ├─ DB Query (18ms)
│  └─ Inventory Service (22ms)
│     └─ Cache Lookup (1ms)
└─ Total: 82ms
```

Traces are invaluable for understanding *where* latency comes from in a distributed system. Without them, you're guessing which service is slow.

### Correlation IDs

A unique identifier (usually a UUID) generated at the entry point of a request and propagated through every service call, log entry, and metric. This lets you connect a user's complaint ("my order failed at 2:30 PM") to the exact trace, logs, and metrics across all services.

### Signal Frameworks

**RED Method** (request-focused — good for services):
- **R**ate: requests per second
- **E**rrors: failed requests per second
- **D**uration: latency distribution

**USE Method** (resource-focused — good for infrastructure):
- **U**tilization: how busy is the resource (CPU, memory, disk, network)
- **S**aturation: how much queued work is waiting
- **E**rrors: error events

**Golden Signals** (Google SRE):
- Latency, Traffic, Errors, Saturation — combines elements of both RED and USE

Pick one framework and apply it consistently. Which one matters less than being consistent.

---

## Load Testing and Capacity Planning

### Load Testing

Simulate realistic traffic patterns to find breaking points *before* production does:

```yaml
# k6 load test example
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up to 100 users
    { duration: '5m', target: 100 },   // stay at 100
    { duration: '2m', target: 500 },   // spike to 500
    { duration: '5m', target: 500 },   // stay at 500
    { duration: '2m', target: 0 },     // ramp down
  ],
};

export default function () {
  const res = http.get('https://api.example.com/orders');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**Types of load tests**:
- **Load test**: expected traffic levels — does the system handle normal load?
- **Stress test**: beyond expected levels — where does it break?
- **Soak test**: sustained load over hours/days — do memory leaks or connection pool exhaustion appear?
- **Spike test**: sudden traffic surge — does the system recover gracefully?

### Capacity Planning

- **Measure current usage**: CPU, memory, disk I/O, network, connection pools
- **Project growth**: based on traffic trends, planned features, marketing campaigns
- **Add headroom**: plan for 2–3x current peak, not average
- **Test the plan**: load test at projected capacity to validate your estimates

---

## Graceful Degradation and Feature Flags

### Graceful Degradation

When part of the system fails, serve a degraded experience instead of a complete failure:

- **Recommendation service down?** Show popular items instead of personalized recommendations
- **Payment provider slow?** Queue the payment and confirm later instead of timing out
- **Search service down?** Show cached results or category browsing instead of returning an error

The key principle: **partial functionality is almost always better than total failure.**

### Feature Flags

Feature flags let you decouple deployment from release and provide a kill switch for new features:

```python
if feature_flags.is_enabled("new_checkout_flow", user_id=user.id):
    return new_checkout(cart)
else:
    return legacy_checkout(cart)
```

**Uses for reliability**:
- **Kill switch**: disable a feature instantly without redeploying if it's causing problems
- **Gradual rollout**: deploy to 1% of users, then 10%, then 50%, then 100%
- **Circuit breaker**: automatically disable a feature when error rates exceed a threshold
- **Load shedding**: disable non-critical features under extreme load to protect core functionality

---

## Multi-Region Deployment

Running your system in multiple geographic regions provides higher availability and lower latency — at significant complexity cost.

### Benefits

- **Latency**: serve users from the nearest region (100ms vs 300ms matters for UX)
- **Availability**: survive an entire region going down (cloud providers have region-level outages)
- **Compliance**: keep data in the region where it was generated (GDPR, data sovereignty)

### Challenges

- **Data replication**: how do you keep data consistent across regions? Sync replication adds latency; async replication means temporary inconsistency
- **Conflict resolution**: if two regions accept writes to the same record, who wins? Last-write-wins, vector clocks, CRDTs — each has trade-offs
- **Operational complexity**: deployments, monitoring, incident response, database migrations — everything is harder with multiple regions
- **Cost**: more infrastructure, more network traffic, more engineering time

### Decision Framework

| Need | Single Region | Multi-Region |
|---|---|---|
| Availability target | 99.9% achievable | 99.99%+ requires it |
| User distribution | Regional | Global |
| Data latency tolerance | Higher | Low latency critical |
| Team size/expertise | Smaller teams | Needs operational maturity |

**Practical advice**: most systems don't need multi-region. Start with a single region, good backups, and fast restore procedures. Only go multi-region when your SLO demands it or your users are truly global.

---

## Exercises

1. **SLO definition**: you're running a REST API for a mobile app. Define 3 SLIs (availability, latency, error rate) with specific SLOs. Calculate the error budget for each over a 30-day window. How many minutes of downtime does each allow?

2. **Postmortem writing**: recall a production incident you've experienced (or find a public postmortem from a company like Google, Cloudflare, or GitHub). Write a blameless postmortem with: timeline, impact, root cause, contributing factors, and 3 concrete action items.

3. **Observability setup**: for a service you work on, verify that logs, metrics, and traces all share a correlation ID. Trace a single request from entry to response. Can you see it across all three signals? If not, identify the gaps.

4. **Load test design**: write a load test for a service you maintain using k6, Locust, or Artillery. Start at current production traffic, ramp to 3x, and identify the first bottleneck. What breaks first: CPU, memory, database connections, or something else?

5. **Graceful degradation audit**: pick 3 external dependencies your system relies on (a database, a third-party API, a cache). For each, answer: what happens to users if this dependency becomes unavailable for 5 minutes? Design a degradation strategy for each.

---

## Recommended Resources

- **"Site Reliability Engineering" by Google (free online: [sre.google/sre-book](https://sre.google/sre-book))** — the foundational text on SLOs, error budgets, and operational excellence.
- **"Implementing Service Level Objectives" by Alex Hidalgo** — practical, detailed guide to choosing and implementing SLOs.
- **"Observability Engineering" by Charity Majors, Liz Fong-Jones, George Miranda** — the definitive book on modern observability practices.
- **Incident.io Blog** — excellent articles on incident response, postmortems, and on-call practices.
- **k6 documentation ([k6.io/docs](https://k6.io/docs))** — well-written docs with practical load testing examples and patterns.
- **Google SRE Workbook (free online)** — companion to the SRE book with hands-on exercises and case studies.
