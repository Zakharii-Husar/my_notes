# Networking

Every application you build communicates over a network. Whether it's a browser fetching a page, a microservice calling another, or a database syncing replicas — networking is the substrate. Understanding how data travels from one machine to another, what can go wrong, and how protocols handle it will make you a dramatically better engineer. You'll debug faster, design more resilient systems, and make informed decisions about latency, throughput, and reliability.

---

## The Network Layer Model

Network communication is organized into layers. Each layer has a specific responsibility and talks to the layers directly above and below it. The most practical mental model maps to four key layers:

### L2 — Data Link (Ethernet)

- Handles communication between devices on the **same local network** (LAN)
- Uses **MAC addresses** (e.g., `aa:bb:cc:dd:ee:ff`) to identify devices
- Ethernet frames carry data between your machine and the nearest switch/router
- You rarely interact with L2 directly, but it matters for understanding local network behavior, ARP, and VLANs

### L3 — Network (IP)

- Handles **routing** packets across networks (your LAN → the internet → destination)
- Uses **IP addresses** (IPv4: `192.168.1.1`, IPv6: `2001:db8::1`)
- Key concepts:
  - **Routing**: routers examine destination IPs and forward packets hop-by-hop
  - **NAT (Network Address Translation)**: allows many devices to share one public IP; your home router does this
  - **Subnets and CIDR**: `10.0.0.0/24` means "the first 24 bits are the network; the last 8 bits identify hosts"
- IP is **unreliable by design** — packets can be lost, duplicated, or arrive out of order

### L4 — Transport (TCP / UDP)

- Adds **ports** to distinguish applications on the same host (e.g., port 80 for HTTP, port 443 for HTTPS)
- **TCP**: reliable, ordered, connection-oriented (the workhorse of the internet)
- **UDP**: unreliable, unordered, connectionless (used for DNS, video streaming, gaming)
- A connection is identified by the 4-tuple: `(src_ip, src_port, dst_ip, dst_port)`

### L7 — Application (HTTP, DNS, TLS, etc.)

- The protocols your application code directly uses
- Built on top of TCP or UDP
- Examples: HTTP, DNS, TLS, WebSocket, gRPC

### Why Layers Matter

When debugging, layers help you isolate problems:

- Can't reach the server at all? → Likely L3 (routing, firewall, DNS)
- Connection established but data is garbled or slow? → Likely L4 (TCP issues)
- Getting HTTP 500 errors? → L7 (application logic)

```bash
# Useful diagnostic tools by layer
ping 8.8.8.8            # L3 — can I reach this IP?
traceroute example.com   # L3 — what path do packets take?
nc -zv example.com 443   # L4 — is the port open?
curl -v https://example.com  # L7 — full HTTP conversation
```

---

## TCP Essentials

TCP is the foundation of most internet communication. It turns the unreliable IP layer into a reliable, ordered byte stream.

### The 3-Way Handshake

Every TCP connection starts with a handshake:

```
Client              Server
  |--- SYN ---------->|     1. Client sends SYN (synchronize)
  |<-- SYN-ACK -------|     2. Server responds with SYN-ACK
  |--- ACK ---------->|     3. Client confirms with ACK
  |                    |     Connection established — data can flow
```

- This takes **one round-trip time (RTT)** before any data is sent
- For HTTPS, add the TLS handshake on top — another 1-2 RTTs
- This is why **connection reuse** (keep-alive, connection pooling) matters so much

### Congestion Control and Slow Start

TCP doesn't blast data at full speed immediately. It **probes** the network capacity:

- **Slow start**: begins with a small congestion window (cwnd), typically 10 segments (~14 KB). Doubles the window each RTT until it detects loss
- **Congestion avoidance**: after the threshold, growth becomes linear (additive increase)
- **On packet loss**: the window is cut dramatically (multiplicative decrease) — this is **AIMD** (Additive Increase, Multiplicative Decrease)

**Implication**: a fresh TCP connection is slow. Transferring a 100 KB response on a new connection takes multiple RTTs just to ramp up. This is why HTTP/2 multiplexing over a single connection is such a win.

### Retransmissions

When a packet is lost (no ACK received within a timeout), TCP retransmits it. Mechanisms:

- **RTO (Retransmission Timeout)**: exponential backoff if no ACK arrives
- **Fast Retransmit**: if 3 duplicate ACKs are received, retransmit immediately without waiting for the timeout
- **SACK (Selective Acknowledgment)**: lets the receiver report exactly which segments it has, so only the missing ones are retransmitted

### Performance Pitfalls

**Head-of-Line (HOL) Blocking**: TCP guarantees in-order delivery. If packet #5 is lost, packets #6, #7, #8 must wait in the receive buffer even though they've arrived. The application is blocked until #5 is retransmitted. This is a major motivation for HTTP/3 (QUIC), which uses UDP and handles ordering per-stream.

**Nagle's Algorithm**: TCP buffers small writes and combines them into larger segments to reduce overhead. This is efficient for bulk transfers but adds latency for interactive protocols. If you send a small request and wait for a response, Nagle can delay the send by up to 200ms waiting for more data.

```python
# Disable Nagle's algorithm when latency matters
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

**Slow Start penalty**: every new connection starts slow. For short-lived connections transferring small payloads, most of the time is spent in handshakes and slow start — not actual data transfer.

---

## HTTP

HTTP is the application-layer protocol that powers the web and most APIs.

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 (QUIC) |
|---|---|---|---|
| Transport | TCP | TCP | UDP (QUIC) |
| Multiplexing | No (one request per connection) | Yes (streams over one connection) | Yes (streams, no HOL blocking) |
| Header compression | No | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Connection setup | TCP + TLS (2-3 RTTs) | TCP + TLS (2-3 RTTs) | 0-1 RTT (TLS built into QUIC) |

- **HTTP/1.1** suffers from HOL blocking at the HTTP level: browsers open 6 parallel connections to work around it
- **HTTP/2** solves HTTP-level HOL blocking with multiplexing but still suffers from TCP-level HOL blocking (one lost packet stalls all streams)
- **HTTP/3** eliminates TCP-level HOL blocking by using QUIC (UDP-based) where each stream is independently reliable

### Caching Headers

Caching is the single biggest performance optimization for HTTP. Master these headers:

**Cache-Control** — the primary directive:

```http
Cache-Control: public, max-age=3600      # Any cache can store; valid for 1 hour
Cache-Control: private, no-cache         # Only browser cache; must revalidate every time
Cache-Control: no-store                  # Never cache (sensitive data)
Cache-Control: stale-while-revalidate=60 # Serve stale while fetching fresh in background
```

**ETag** — content-based validation:

```http
# Response
HTTP/1.1 200 OK
ETag: "abc123"

# Subsequent request — conditional
GET /resource HTTP/1.1
If-None-Match: "abc123"

# If unchanged:
HTTP/1.1 304 Not Modified    # No body sent — saves bandwidth
```

**Vary** — tells caches that the response depends on specific request headers:

```http
Vary: Accept-Encoding, Accept-Language
```

Without `Vary`, a cache might serve a gzip-compressed response to a client that doesn't support gzip, or an English response to a French-speaking user.

### Content Negotiation and Compression

- **Accept / Content-Type**: client and server negotiate the response format (JSON, HTML, XML)
- **Accept-Encoding**: client declares supported compression (`gzip`, `br` for Brotli, `zstd`)
- **Transfer-Encoding: chunked**: server sends the response in chunks without knowing the total size upfront — useful for streaming or dynamically generated content

```http
GET /api/data HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, br
```

### Idempotency, Retries, and Backoff

**Idempotency** means making the same request multiple times produces the same result. This is critical for safe retries:

| Method | Idempotent? | Safe? |
|--------|-------------|-------|
| GET    | Yes         | Yes   |
| PUT    | Yes         | No    |
| DELETE | Yes         | No    |
| POST   | **No**      | No    |

For non-idempotent operations, use an **idempotency key**:

```http
POST /payments HTTP/1.1
Idempotency-Key: txn-abc-123
Content-Type: application/json

{"amount": 100, "currency": "USD"}
```

The server stores the result keyed by `txn-abc-123`. If the client retries (e.g., after a timeout), the server returns the stored result instead of processing a duplicate payment.

**Retry with exponential backoff and jitter**:

```python
import time, random

def retry_with_backoff(fn, max_retries=5, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return fn()
        except RetryableError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.5)
            time.sleep(delay + jitter)
```

**Timeouts**: always set them. A missing timeout means a failed server can hold your connection indefinitely:

- **Connect timeout**: how long to wait to establish the TCP connection (typically 1-5s)
- **Read timeout**: how long to wait for a response after sending the request (varies by endpoint)

---

## DNS

DNS translates human-readable names (`example.com`) into IP addresses (`93.184.216.34`). It's a distributed, hierarchical database.

### How Resolution Works

```
Your app → Stub resolver (OS) → Recursive resolver (ISP / 8.8.8.8)
                                    → Root nameserver (.)
                                    → TLD nameserver (.com)
                                    → Authoritative nameserver (example.com)
                                    ← IP address returned
```

1. Your app calls `getaddrinfo()` which asks the OS stub resolver
2. The stub resolver checks its local cache, then asks the configured recursive resolver
3. The recursive resolver walks the hierarchy: root → TLD → authoritative
4. Results are cached at every level based on **TTL (Time To Live)**

### Record Types

| Type  | Purpose | Example |
|-------|---------|---------|
| A     | Name → IPv4 address | `example.com → 93.184.216.34` |
| AAAA  | Name → IPv6 address | `example.com → 2606:2800:220:1:...` |
| CNAME | Alias → canonical name | `www.example.com → example.com` |
| MX    | Mail exchange | `example.com → mail.example.com` |
| TXT   | Arbitrary text (SPF, verification) | `example.com → "v=spf1 ..."` |
| NS    | Authoritative nameserver | `example.com → ns1.example.com` |

### Split-Horizon DNS

Returns different answers depending on where the query comes from:

- Internal users → private IPs (`10.0.1.5`)
- External users → public IPs (`203.0.113.50`)

Used in corporate networks and cloud environments to route internal traffic privately.

### Common DNS Issues

- **Propagation delay**: after changing a DNS record, old values remain cached until TTL expires across resolvers worldwide. There's no way to force a global flush
- **Stale records**: if you lower TTL before a migration, some resolvers ignore TTL floors or cache aggressively
- **TTL trade-off**: low TTL = fast failover but more DNS queries; high TTL = fewer queries but slower failover

```bash
# Inspect DNS records and TTL
dig example.com A +short
dig example.com A +noall +answer    # shows TTL
nslookup example.com 8.8.8.8        # query a specific resolver
```

---

## TLS

TLS (Transport Layer Security) encrypts communication between client and server. Without it, anyone on the network path can read and modify your traffic.

### Certificates and Chains

- A **certificate** binds a public key to a domain name, signed by a Certificate Authority (CA)
- **Chain of trust**: your cert is signed by an intermediate CA, which is signed by a root CA. Browsers/OS trust a set of root CAs
- If you're missing an intermediate certificate in your chain, some clients will fail to verify even though your cert is valid

```
Root CA (trusted by OS/browser)
  └── Intermediate CA
        └── Your server certificate (example.com)
```

### Key TLS Concepts

**SNI (Server Name Indication)**: the client sends the requested hostname in the TLS handshake (in plaintext). This lets a single IP host multiple HTTPS sites. Without SNI, the server wouldn't know which certificate to present.

**ALPN (Application-Layer Protocol Negotiation)**: during the TLS handshake, client and server agree on the application protocol (e.g., `h2` for HTTP/2, `http/1.1`). This avoids an extra round trip for protocol negotiation.

**Forward Secrecy (PFS)**: even if the server's private key is compromised in the future, past recorded traffic cannot be decrypted. Achieved using ephemeral key exchange (ECDHE). Each session generates a unique key that is discarded afterward.

### mTLS (Mutual TLS)

In standard TLS, only the server proves its identity. In **mTLS**, the client also presents a certificate:

```
Client                          Server
  |--- ClientHello ------------->|
  |<-- ServerHello + ServerCert -|
  |--- ClientCert + Verify ----->|   ← Client also authenticates
  |<-- Finished -----------------|
```

- Common in **service-to-service** communication (service mesh, internal APIs)
- Each service gets its own certificate, often issued by an internal CA
- Provides strong identity — no passwords or API keys needed
- Tools like Istio, Linkerd, and SPIFFE automate mTLS certificate management

---

## WebSocket and SSE

HTTP is request-response: the client asks, the server answers. But many applications need real-time server-initiated communication.

### WebSocket

- Starts as an HTTP upgrade request, then switches to a **full-duplex binary protocol**
- Both client and server can send messages at any time
- Maintains a persistent TCP connection

```javascript
// Client-side WebSocket
const ws = new WebSocket('wss://example.com/stream');

ws.onopen = () => {
    ws.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));
};

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Received:', data);
};

ws.onclose = (event) => {
    console.log(`Connection closed: code=${event.code}, reason=${event.reason}`);
    // Implement reconnection logic with backoff
};
```

**When to use WebSocket**: chat, collaborative editing, multiplayer games, live trading — any scenario requiring bidirectional, low-latency communication.

**Considerations**:
- WebSocket connections are stateful — they don't work well with simple HTTP load balancers (need sticky sessions or L4 balancing)
- You need to handle reconnection, heartbeats, and message ordering yourself
- Scaling requires careful connection management

### Server-Sent Events (SSE)

- Server → client only (unidirectional)
- Uses a standard HTTP response with `Content-Type: text/event-stream`
- Built on top of regular HTTP — works with existing infrastructure (proxies, load balancers, CDNs)
- Automatic reconnection built into the browser API

```python
# Server-side SSE (Python / Flask-style)
from flask import Response

def event_stream():
    while True:
        data = get_latest_update()
        yield f"id: {data['id']}\nevent: update\ndata: {json.dumps(data)}\n\n"

@app.route('/events')
def stream():
    return Response(event_stream(), content_type='text/event-stream')
```

```javascript
// Client-side SSE
const source = new EventSource('/events');

source.addEventListener('update', (event) => {
    const data = JSON.parse(event.data);
    console.log('Update:', data);
});

source.onerror = () => {
    // Browser automatically reconnects
    console.log('Connection lost, reconnecting...');
};
```

### WebSocket vs SSE — Decision Guide

| Criteria | WebSocket | SSE |
|----------|-----------|-----|
| Direction | Bidirectional | Server → Client |
| Protocol | Custom (over TCP) | Standard HTTP |
| Reconnection | Manual | Automatic |
| Binary data | Yes | No (text only) |
| Infrastructure | Needs WS-aware proxies | Works everywhere HTTP works |
| Complexity | Higher | Lower |

**Rule of thumb**: if you only need server-to-client updates (dashboards, notifications, feeds), use SSE. It's simpler, HTTP-friendly, and "just works." Reach for WebSocket when you genuinely need bidirectional communication.

---

## Exercises

1. **Packet trace**: use `curl -v https://example.com` and identify the TCP handshake, TLS handshake, HTTP request/response, and connection close in the output. How many round trips did it take before the first byte of content?

2. **DNS investigation**: using `dig`, find the full resolution chain for a domain you use daily. What's the TTL? What happens if you query different recursive resolvers (e.g., `8.8.8.8` vs `1.1.1.1`)? Do you get different answers?

3. **Caching headers**: inspect the response headers of 5 different websites using `curl -I`. Which use `ETag`? Which use `Cache-Control`? Do any use `Vary`? Can you identify which resources are cacheable and for how long?

4. **Idempotency design**: you're building a payment API. Design the flow for safely retrying a failed payment request. What headers and server-side storage do you need? What happens if the server processes the payment but the response is lost?

5. **WebSocket vs SSE**: you're building a live sports score dashboard. Users watch scores update in real-time but never send messages. Which protocol would you choose and why? What changes if users can also post comments?

6. **TLS certificate chain**: run `openssl s_client -connect example.com:443 -showcerts` and identify the leaf certificate, intermediate(s), and root CA. What happens if the server doesn't send the intermediate cert?

---

## Recommended Resources

- **"High Performance Browser Networking" by Ilya Grigorik** — freely available at [hpbn.co](https://hpbn.co). The definitive guide to TCP, TLS, HTTP/2, and WebSocket performance. Start here.
- **Beej's Guide to Network Programming** — hands-on socket programming in C. Great for building intuition about what's really happening at the OS level.
- **RFC 9110 (HTTP Semantics)** — the authoritative reference for HTTP methods, headers, and status codes. Dense but precise.
- **Cloudflare Learning Center** — excellent, accessible articles on DNS, TLS, and HTTP/3/QUIC.
- **Julia Evans' networking zines** — visual, concise explanations of DNS, HTTP, and networking concepts. Perfect for quick reference.
- **Wireshark** — capture and inspect real network traffic. Nothing builds understanding faster than seeing actual packets.
