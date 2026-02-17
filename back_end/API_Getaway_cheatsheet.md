# API Gateway — Cheat Sheet

## What an API Gateway is

An **API Gateway** is a **reverse proxy / “front door”** that sits **in front of one or many backend services** and provides a **single public entry point** for clients.

**Client → API Gateway (routing + policies) → backend service(s)**

Instead of clients calling multiple services directly, they call one domain (e.g. `api.example.com`) and the gateway forwards requests to the right backend.

---

## What it does (common responsibilities)

### Routing / reverse proxy

- Path-based routing: `/users/*` → users service, `/orders/*` → orders service
- Host-based routing: `admin.api.com` → admin service
- Version routing: `/v1/*` vs `/v2/*`

### Security & auth “at the edge”

- JWT validation, OAuth2/OIDC integration
- API keys, HMAC signatures
- IP allow/deny rules, WAF integration (depends on product)

### Rate limiting & quotas

- Per-IP, per-user, per-key throttling
- Usage plans (e.g. “free tier 100 req/min, pro 1000 req/min”)

### Request/response transforms

- Header injection/removal, path rewriting
- Normalizing responses (product-dependent)

### Observability

- Centralized logs/metrics/tracing
- Correlation IDs, request logging

### Reliability features (depends on product)

- Timeouts, retries
- Circuit breaker patterns (more common with service mesh/Envoy stacks)
- Caching (some gateways support it)

---

## What it is NOT

- Not just “a list of endpoints”
- Not simply “bundling ports”
- Not required for a single monolithic API (often a load balancer/reverse proxy is enough)

---

## In a monolith: what you usually have instead

If you have **one monolithic CRUD + WebSocket backend**, your “gateway layer” is typically just:

- **Load balancer** and/or **reverse proxy** in front of the app (TLS termination + forwarding)

Common options:

- Nginx / Caddy / Traefik
- Cloud load balancer (AWS ALB, etc.)
- Cloudflare in front of your server (edge protection + routing features)

This often gives you:

- Single domain (`api.example.com`)
- HTTPS/TLS termination
- Basic routing/headers
- Sometimes basic rate limiting

---

## When you NEED an API Gateway (practical triggers)

### 1) Multiple backends, one public API

You have multiple systems behind one domain:

- REST API (Go)
- WebSocket server separate
- Serverless endpoints (webhooks, background processing)
- Separate admin service

You want:

- `api.example.com/*` → app
- `/webhooks/stripe` → webhook handler
- `/ws` → WS service (or separate domain)

### 2) You want edge policies without re-implementing in every service

- Central auth checks (JWT/OIDC) before traffic hits services
- API keys for partners
- Throttling/quotas
- Consistent request validation, logging

### 3) Multiple client types with different rules

- Public API vs internal admin API
- Partner API with stricter keys/limits
- Mobile API rules vs web rules

### 4) Versioning / canary routing without changing clients

- Route `/v1` to old, `/v2` to new
- Gradual rollout (where supported) without updating client config

### 5) Kubernetes / microservices

Standard pattern:

- Keep services private inside the cluster
- Gateway is the controlled public ingress point

---

## When you DO NOT need it

You likely don’t need a dedicated API Gateway if you have:

- **One backend service**
- **One auth system**
- **No partner API keys/usage plans**
- **No complex routing/versioning**
- **No need for centralized policies beyond what your app already handles**

In that case:

- Use a reverse proxy/load balancer + app-level auth
- Add a gateway only when the overhead becomes worth it

---

## WebSockets note

A gateway is **not required** for WebSockets, but you still need a front layer that supports:

- HTTP Upgrade
- Reasonable timeouts / keepalive
- (Sometimes) sticky sessions

This can be:

- Nginx/Traefik/Cloudflare
- A cloud load balancer that supports WebSockets
- Some API gateways (support varies; many prefer keeping WS behind a load balancer)

---

## API Gateway “as a Service” (managed gateways)

Cloud providers offer managed API gateways that you provision (often via Terraform):

Examples:

- **AWS API Gateway**
- **Azure API Management**
- **Google API Gateway / Apigee**

Typically managed gateways handle:

- Domains + TLS certs
- Routes + integrations to backends (LBs, containers, serverless)
- Auth/authorizers
- Rate limits / usage plans
- Logging and environments (dev/stage/prod)

Terraform is common because the config is large and should be reproducible:

- routes, stages, auth, quotas, logging, deployments

---

## Self-hosted gateways (run it yourself)

You can run your own gateway/proxy when you want full control or avoid managed limitations:

Examples:

- Kong, NGINX, Traefik, Envoy, Tyk, Ambassador
- “Ingress Gateway” patterns in Kubernetes (often Envoy-based)

---

## Simple mental model

**API Gateway = routing + policies at the edge.**

If your architecture is simple, a **reverse proxy + your monolith** is usually enough.
Add a real gateway when you have **multiple backends** or you need **centralized control** (auth, quotas, partner access, version routing, observability).
