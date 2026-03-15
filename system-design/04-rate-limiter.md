# Design a Rate Limiter

A rate limiter controls the rate of traffic a client can send. If the request count exceeds a threshold within a time window, excess requests are **throttled** (dropped or queued). Essential for preventing abuse, controlling cost, and protecting downstream services.

---

## Table of Contents

1. [Why Rate Limiting?](#why-rate-limiting)
2. [Where to Put the Rate Limiter](#where-to-put-the-rate-limiter)
3. [Rate Limiting Algorithms](#rate-limiting-algorithms)
4. [High-Level Architecture](#high-level-architecture)
5. [Detailed Design](#detailed-design)
6. [Rate Limiting Rules](#rate-limiting-rules)
7. [HTTP Headers & Client Behavior](#http-headers--client-behavior)
8. [Distributed Rate Limiting](#distributed-rate-limiting)
9. [Race Conditions & Solutions](#race-conditions--solutions)
10. [Monitoring & Operational Concerns](#monitoring--operational-concerns)
11. [Background: Client-Side Rate Limiting](#background-client-side-rate-limiting)
12. [Background: Gateway vs Server-Side Rate Limiting](#background-gateway-vs-server-side-rate-limiting)
13. [Background: API Gateway, Middleware & SSL Termination](#background-api-gateway-middleware--ssl-termination)

---

## Why Rate Limiting?

```
Without rate limiting:                 With rate limiting:

  Attacker ──→ 100K req/s ──→ DB     Attacker ──→ 100K req/s ──→ [Rate Limiter]
                          💥 DOWN                                      │
                                                              ┌───────┴──────┐
                                                              │ 100 req/s    │ 99.9K dropped
                                                              ▼              │ (429 Too Many)
                                                             DB (healthy)    ▼
                                                                          Rejected
```

### Use Cases

| Purpose | Example |
|---------|---------|
| **Prevent DoS** | Limit requests per IP/user |
| **Cost control** | Cap 3rd-party API calls (avoid surprise bills) |
| **Fair usage** | Free tier: 100 req/min, paid: 10K req/min |
| **Protect downstream** | Database can handle 5K QPS, limit at 4K |
| **Bot prevention** | Limit login attempts, scraping |

---

## Where to Put the Rate Limiter

### Three Options

```
Option 1: Client-Side                Option 2: Server-Side (Middleware)
┌────────┐                           ┌────────┐    ┌─────────────┐    ┌────────┐
│ Client │─── self-throttle          │ Client │──>│Rate Limiter │──>│  API   │
└────────┘                           └────────┘    │ (middleware) │    │Server  │
  ✗ Easily bypassed                  └────────┘    └─────────────┘    └────────┘
  ✗ Can't trust client                              ✓ Most common

Option 3: Dedicated Service (API Gateway)
┌────────┐    ┌──────────────────────────┐    ┌────────┐
│ Client │──>│     API Gateway           │──>│  API   │
└────────┘    │ (rate limit + auth +     │    │Server  │
              │  SSL + routing)          │    └────────┘
              └──────────────────────────┘
               ✓ Cloud-managed (AWS API GW, Kong, Envoy)
```

### Decision Guide

```
┌──────────────────────────────────────────────────────────────┐
│ If you already have an API gateway  →  add rate limiting there│
│ If building from scratch            →  server-side middleware │
│ If microservices architecture       →  API gateway or sidecar│
│ If you need flexibility/control     →  build your own service│
└──────────────────────────────────────────────────────────────┘
```

---

## Rate Limiting Algorithms

### 1. Token Bucket

```
          ┌─────────────────┐
          │  Token Refiller  │  ← adds tokens at fixed rate
          │  (e.g., 5/sec)  │
          └────────┬────────┘
                   ▼
          ┌─────────────────┐
          │  Bucket          │  ← max capacity (e.g., 10 tokens)
          │  [●][●][●][●]   │
          └────────┬────────┘
                   │
    Request arrives │
                   ▼
          ┌─────────────────┐
          │ Token available? │
          │   Yes → consume  │──→ Forward request ✓
          │   No  → reject   │──→ 429 Too Many ✗
          └─────────────────┘
```

**Parameters:** Bucket size (max burst), Refill rate (sustained throughput)

| Pros | Cons |
|------|------|
| Simple to implement | Two parameters to tune |
| Memory efficient (counter + timestamp) | |
| Allows bursts up to bucket size | |
| Used by Amazon, Stripe | |

**Example:** Bucket size = 10, Refill rate = 5 tokens/sec

```
Timeline:
  Start: 10 tokens
  User sends 7 requests quickly → allowed, 3 tokens left
  User sends 4 more immediately → only 3 allowed, 1 rejected
  Wait 1 second → 5 tokens refilled → more requests can pass

Why better than "exact 5 per second":
  Real traffic is bursty. Users don't send requests perfectly evenly.
  Token bucket handles both steady rate AND burst tolerance.
```

---

### 2. Leaking Bucket

```
    Requests arrive (variable rate)
          │ │ │ │ │ │ │
          ▼ ▼ ▼ ▼ ▼ ▼ ▼
    ┌─────────────────────┐
    │      FIFO Queue      │  ← fixed size (e.g., 5)
    │  [R1][R2][R3][R4][R5]│
    │  [overflow → DROP]   │  ← queue full = reject
    └──────────┬──────────┘
               │
               ▼ processes at fixed rate (e.g., 2/sec)
    ┌─────────────────────┐
    │    API Server        │
    └─────────────────────┘
```

**Parameters:** Bucket (queue) size, Outflow rate

| Pros | Cons |
|------|------|
| Smooth output rate (no bursts) | Burst of traffic fills queue with old requests |
| Memory efficient (fixed queue) | Recent requests may be dropped while old ones wait |
| Stable processing rate | Not ideal when burst tolerance is needed |

---

### 3. Fixed Window Counter

```
Timeline: ──────────────────────────────────────────────────────→

Window 1 (0:00-1:00)          Window 2 (1:00-2:00)
┌─────────────────────┐       ┌─────────────────────┐
│ Counter: 3/5         │       │ Counter: 1/5         │
│ ✓ ✓ ✓               │       │ ✓                    │
└─────────────────────┘       └─────────────────────┘

Request at 0:30 → Counter=4 → ✓ Allow (< 5)
Request at 0:45 → Counter=5 → ✓ Allow (= 5)
Request at 0:50 → Counter=6 → ✗ Reject (> 5)
Request at 1:01 → Counter=1 → ✓ Allow (new window)
```

| Pros | Cons |
|------|------|
| Memory efficient | **Boundary problem**: burst at edge of two windows |
| Simple to understand | can allow 2× the rate at window boundaries |
| Resets at end of window | |

**The Boundary Problem:**

```
Limit: 5 req/min

         Window 1              Window 2
    ├──────────────────┼──────────────────┤
    │                  │                  │
    │         5 req ───┼── 5 req          │
    │     (0:30-1:00)  │ (1:00-1:30)     │
    │                  │                  │

    In the 1-min span from 0:30 → 1:30:
    10 requests went through! (2× the limit)
```

---

### 4. Sliding Window Log

```
Limit: 3 requests per minute

Request log (sorted timestamps):
┌──────────────────────────────────────────────────────────────┐
│  [1:00:01]  [1:00:30]  [1:00:50]  │  [1:01:10]              │
│                                     │                         │
│  Window: current_time - 60s         │                         │
└──────────────────────────────────────┘                        │
                                                                │
New request at 1:01:10:                                         │
  1. Remove entries older than 1:00:10 → remove [1:00:01]       │
  2. Count remaining: [1:00:30, 1:00:50] = 2                    │
  3. 2 < 3 → ✓ Allow, add [1:01:10] to log                     │
```

| Pros | Cons |
|------|------|
| Very accurate — no boundary issues | **High memory** — stores every timestamp |
| Smooth rate limiting | Expensive for high-traffic endpoints |

---

### 5. Sliding Window Counter (Hybrid)

Combines fixed window counter + sliding window math. **Best balance of accuracy and efficiency.**

```
Limit: 7 requests per minute
Previous window (0:00-1:00): 5 requests
Current window  (1:00-2:00): 3 requests (so far)

Current time: 1:15  → 75% into current window → 25% of previous overlaps

Weighted count = (prev × overlap%) + current
               = (5 × 0.75) + 3
               = 3.75 + 3
               = 6.75

6.75 < 7 → ✓ Allow

┌──────────────────┬──────────────────┐
│ Previous Window   │ Current Window    │
│ count = 5         │ count = 3         │
│                   │                   │
│         ┌─────────┼────┐              │
│         │ 75% of  │    │ 25% into     │
│         │ prev    │    │ current      │
│         │ window  │    │              │
│         └─────────┼────┘              │
└──────────────────┴──────────────────┘
         ← sliding window →
```

| Pros | Cons |
|------|------|
| Memory efficient (just 2 counters per window) | Not 100% exact (approximation) |
| Smooths out boundary spikes | Assumes even distribution in prev window |
| Good enough for most production use | |
| Used by Cloudflare | |

---

### Algorithm Comparison

```
┌──────────────────────┬────────┬──────────┬──────────┬──────────┐
│ Algorithm            │ Memory │ Accuracy │ Burst    │ Use When │
├──────────────────────┼────────┼──────────┼──────────┼──────────┤
│ Token Bucket         │ Low    │ Good     │ ✓ Yes    │ API rate │
│                      │        │          │ (burst   │ limiting │
│                      │        │          │  ok)     │ (most    │
│                      │        │          │          │ common)  │
├──────────────────────┼────────┼──────────┼──────────┼──────────┤
│ Leaking Bucket       │ Low    │ Good     │ ✗ No     │ Smooth   │
│                      │        │          │ (smooths │ output   │
│                      │        │          │  out)    │ needed   │
├──────────────────────┼────────┼──────────┼──────────┼──────────┤
│ Fixed Window Counter │ Low    │ Medium   │ ⚠ Edge   │ Simple   │
│                      │        │          │ burst    │ use case │
├──────────────────────┼────────┼──────────┼──────────┼──────────┤
│ Sliding Window Log   │ High   │ Perfect  │ ✓ Yes    │ Need     │
│                      │        │          │          │ precision│
├──────────────────────┼────────┼──────────┼──────────┼──────────┤
│ Sliding Window       │ Low    │ High     │ ✓ Yes    │ Best     │
│ Counter              │        │ (approx) │          │ balance  │
└──────────────────────┴────────┴──────────┴──────────┴──────────┘
```

---

## High-Level Architecture

```
┌────────┐     ┌──────────────────────────────────────┐     ┌────────────┐
│ Client │────>│          Rate Limiter Middleware       │────>│  API Server│
└────────┘     │                                       │     └────────────┘
               │  1. Identify client (IP, user, API key)│
               │  2. Fetch rules from config             │
               │  3. Check counter in Redis              │
               │  4. Allow or reject (429)               │
               └─────────────┬─────────────────────────┘
                             │
                    ┌────────┴────────┐
                    │     Redis        │
                    │  (counters/      │
                    │   timestamps)    │
                    └─────────────────┘
```

### Why Redis?

```
✓ In-memory → microsecond latency
✓ INCR + EXPIRE are atomic operations
✓ Supports TTL natively (auto-cleanup)
✓ Shared across multiple API servers
✓ Battle-tested for rate limiting
```

### Request Flow

```
Client sends request
       │
       ▼
┌──────────────────┐
│ Rate limiter      │
│ middleware         │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐    ┌────────────────────────┐
│ Fetch rate limit  │───>│ Rules config           │
│ rules for client  │    │ (YAML/DB/config svc)   │
└──────┬───────────┘    └────────────────────────┘
       │
       ▼
┌──────────────────┐    ┌────────────────────────┐
│ Check counter     │◄──>│ Redis                  │
│ in Redis          │    │ key: rate_limit:{id}   │
└──────┬───────────┘    │ val: request_count     │
       │                 │ TTL: window_size       │
       │                 └────────────────────────┘
       ▼
┌──────────────────┐
│ Counter ≤ limit? │
│   Yes → forward  │──→ API Server → 200 OK
│   No  → reject   │──→ 429 Too Many Requests
└──────────────────┘
```

---

## Detailed Design

### Counter Storage in Redis (Token Bucket)

```
Key:    rate_limit:{client_id}
Value:  { tokens: 4, last_refill: 1678901234 }
TTL:    auto-expire after inactivity

On each request:
  1. Get current token count
  2. Calculate tokens to add since last_refill
  3. tokens = min(tokens + added, bucket_size)
  4. If tokens > 0 → decrement, allow
  5. If tokens = 0 → reject
```

### Counter Storage in Redis (Fixed/Sliding Window)

```
Key:    rate_limit:{client_id}:{window_start}
Value:  counter (integer)
TTL:    window_size (auto-cleanup)

INCR rate_limit:user123:1678901200    # atomic increment
EXPIRE rate_limit:user123:1678901200 60  # 60-sec TTL

If counter > limit → reject
```

---

## Rate Limiting Rules

Rules are typically stored in config files or a config service and define **who, what, and how much**.

```yaml
# Example rules (YAML config)
rules:
  - domain: messaging
    descriptors:
      - key: message_type
        value: marketing
        rate_limit:
          requests_per_unit: 5
          unit: day

  - domain: auth
    descriptors:
      - key: endpoint
        value: /api/login
        rate_limit:
          requests_per_unit: 5
          unit: minute

  - domain: api
    descriptors:
      - key: tier
        value: free
        rate_limit:
          requests_per_unit: 100
          unit: minute
      - key: tier
        value: premium
        rate_limit:
          requests_per_unit: 10000
          unit: minute
```

---

## HTTP Headers & Client Behavior

### Response Headers

```
HTTP/1.1 200 OK
X-Ratelimit-Remaining: 42       ← requests left in current window
X-Ratelimit-Limit: 100          ← max requests per window
X-Ratelimit-Retry-After: 30     ← seconds until window resets (on 429)
```

### 429 Response

```
HTTP/1.1 429 Too Many Requests
X-Ratelimit-Remaining: 0
X-Ratelimit-Limit: 100
X-Ratelimit-Retry-After: 30
Content-Type: application/json

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Retry after 30 seconds."
}
```

### What Happens to Dropped Requests?

```
Option 1: Drop immediately → return 429 (most common)
Option 2: Queue for later  → message queue → process when capacity available
Option 3: Degrade gracefully → serve cached/stale response

┌─────────────┐
│   Rejected   │
│   Request    │
└──────┬──────┘
       │
  ┌────┴────┐
  │ Config  │
  ├─────────┤
  │ drop    │──→ 429 (most common)
  │ queue   │──→ Message Queue → delayed processing
  │ degrade │──→ Return cached/partial response
  └─────────┘
```

---

## Distributed Rate Limiting

### The Problem

Multiple rate limiter instances need a **shared counter** — otherwise each instance allows the full limit independently.

```
Without shared state:                 With shared Redis:

Server A: 100/100 ✓                  Server A ──┐
Server B: 100/100 ✓                              ├──→ Redis: 100/100
Server C: 100/100 ✓                  Server B ──┤
                                                  ├──→ (shared counter)
Total: 300 requests!                 Server C ──┘
(3× the intended limit)
                                     Total: 100 requests ✓
```

### Why Distributed is Needed

```
If you have 5 API servers behind a load balancer:
  - You want: user A allowed 100 req/min TOTAL
  - Not: 100 req/min PER SERVER (= 500 total)
  - So counters must be shared (Redis)
```

### Synchronization Strategies

```
┌────────────────────────────────────────────────────────────────┐
│ Strategy              │ How                    │ Trade-off      │
├───────────────────────┼────────────────────────┼────────────────┤
│ Centralized Redis     │ All servers share one  │ Simple, but    │
│                       │ Redis for counters     │ Redis = SPOF   │
│                       │                        │                │
│ Redis Cluster         │ Sharded Redis with     │ Scalable,      │
│                       │ replication            │ more complex   │
│                       │                        │                │
│ Sticky sessions       │ Route same client to   │ Less accurate, │
│                       │ same server            │ uneven load    │
│                       │                        │                │
│ Local + sync          │ Local counters, sync   │ Eventually     │
│                       │ periodically to Redis  │ consistent     │
└───────────────────────┴────────────────────────┴────────────────┘
```

---

## Race Conditions & Solutions

### The Race Condition

```
Time    Server A                    Redis               Server B
─────────────────────────────────────────────────────────────────
 t1     GET counter → 3
 t2                                                     GET counter → 3
 t3     3 < 5, allow                                    3 < 5, allow
 t4     SET counter = 4             counter = 4
 t5                                 counter = 4         SET counter = 4
                                    ↑ WRONG! Should be 5

Both read 3, both increment to 4 → lost update!
```

### Solution 1: Lua Script (Atomic)

```lua
-- Atomic check-and-increment in Redis
local current = redis.call('GET', KEYS[1])
if current and tonumber(current) >= tonumber(ARGV[1]) then
    return 0  -- rejected
end
redis.call('INCR', KEYS[1])
if tonumber(current) == nil then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
return 1  -- allowed
```

> Lua scripts execute **atomically** in Redis — no interleaving.

### Solution 2: Redis INCR (Already Atomic)

```
INCR rate_limit:user123:window
→ Redis returns new value atomically
→ If value > limit → reject (but request already counted)

Simple and works well. Slight over-count on rejection
(counter incremented even for rejected requests) — usually acceptable.
```

### Solution 3: Sorted Sets (Sliding Window Log)

```
ZADD  rate_limit:user123  <timestamp>  <request_id>
ZREMRANGEBYSCORE rate_limit:user123 0 <timestamp - window>
ZCARD rate_limit:user123
→ If count > limit → reject + ZREM the just-added entry
```

---

## Monitoring & Operational Concerns

### What to Monitor

```
┌────────────────────────────────────────────────────────────────┐
│ Metric                        │ Why                            │
├───────────────────────────────┼────────────────────────────────┤
│ Total requests allowed/sec    │ Baseline traffic               │
│ Total requests rejected/sec   │ Rate limiting effectiveness    │
│ Rejection ratio (%)           │ Too high? Rules too strict     │
│                               │ Too low? Rules too lenient     │
│ Redis latency (p50, p99)      │ Rate limiter adding latency?   │
│ Rules config version          │ Track rule changes             │
│ Per-client rejection rates    │ Identify abusers or bad config │
└───────────────────────────────┴────────────────────────────────┘
```

### Alerting Thresholds

```
⚠ Alert if:
  - Rejection rate > 10% globally (rules may be too aggressive)
  - Rejection rate = 0% for days (rate limiter might be broken)
  - Redis latency p99 > 10ms (degrading request path)
  - Single client > 50% of all rejections (possible attack)
```

### Design Considerations Checklist

```
✓ Hard vs soft rate limiting
    Hard: strict limit, never exceed
    Soft: allow brief bursts above limit

✓ Rate limiting at different levels
    Per user, per IP, per API key, per endpoint, global

✓ Multi-tier limits
    100 req/sec AND 10,000 req/hour (both must pass)

✓ Exempt internal services?
    Service-to-service calls may bypass rate limiting

✓ Graceful degradation
    If Redis is down: fail open (allow all) or fail closed (block all)?
    Usually fail open — rate limiter shouldn't cause outage

✓ Client communication
    Clear 429 responses with Retry-After header
    Documentation on rate limits per tier
```

---

## Background: Client-Side Rate Limiting

### What It Is

Rate limiting done in the frontend/mobile/SDK **before the request is sent**.

```
Examples:
  - Web app disables repeated button clicks
  - SDK sends max 3 req/sec
  - Mobile app queues requests instead of firing too many
```

### Purpose

- Reducing accidental spam
- Improving UX
- Reducing wasted network calls
- Protecting battery/bandwidth

### Why It's Unreliable

```
Client-side rate limiting is ADVISORY, not AUTHORITATIVE.

Users can bypass your frontend entirely:
  - curl / Postman / Python script
  - Browser dev tools
  - Modified mobile app
  - Bot network

  for i in {1..1000}; do
    curl https://api.example.com/endpoint
  done

Rule: Anything enforced only on client side is NOT real security.
      Real enforcement must happen on the server/gateway side.
```

### When Client-Side Is Useful

```
✓ UX improvement (disable button after click)
✓ Reducing accidental duplicate requests
✓ SDK politeness (don't spam the API)
✗ NOT for security enforcement
✗ NOT for abuse prevention
```

---

## Background: Gateway vs Server-Side Rate Limiting

### Gateway Rate Limiter

Runs **before** requests hit your backend services.

```
Browser → HTTPS → [Gateway Rate Limiter] → Backend Service
                         │
                    Blocks traffic early
                    Reduces load on backend
                    Central enforcement
```

**Best for:**
- 100 req/min per IP
- 1000 req/min per API key
- 10 login attempts/min
- DDoS/basic abuse control

### Server-Side Rate Limiter

Runs **inside** the backend application/service.

```
Browser → Gateway → Backend → [Server Rate Limiter] → Business Logic
                                      │
                                 Closer to business logic
                                 Uses business-specific data
```

**Best for:**
- User can create only 3 reports per minute
- Tenant can start only 2 export jobs concurrently
- AI endpoint allows only N tokens/day per account plan

### Best Practice: Use Both

```
┌────────────────────────────────────────────────────────────┐
│ Layer          │ What it protects     │ Examples            │
├────────────────┼──────────────────────┼─────────────────────┤
│ Gateway        │ System edge          │ IP limits, API key  │
│ (coarse)       │ (DDoS, abuse)        │ quotas, auth limits │
├────────────────┼──────────────────────┼─────────────────────┤
│ Server-side    │ Application logic    │ Business rules,     │
│ (fine-grained) │ (specific actions)   │ resource quotas     │
└────────────────┴──────────────────────┴─────────────────────┘

Gateway limiter protects the SYSTEM EDGE.
Server limiter protects the APPLICATION LOGIC.
```

### Rate Limiter Placement in Different Architectures

```
Single server:
  Rate limiter in Nginx / app middleware / Redis

Microservices:
  Rate limiter at API gateway (global protection)
  + inside each service (business limits)
  + shared Redis/quota service (distributed limits)
```

---

## Background: API Gateway, Middleware & SSL Termination

### What Is an API Gateway?

The **front door** to your backend system. Clients call the gateway first; gateway forwards to internal services.

```
Client → API Gateway → Internal Services

Common responsibilities:
  - Routing requests to the right service
  - Authentication / auth checks
  - Rate limiting
  - Logging / metrics
  - SSL termination
  - CORS handling
  - Load balancing
  - API key validation
  - Request/response transformation

Examples: Kong, NGINX, AWS API Gateway, Envoy, Apigee
```

### What Is SSL/TLS Termination?

```
When client uses HTTPS, traffic is encrypted with TLS.
SSL termination = gateway decrypts the HTTPS traffic there.

Browser ──HTTPS (encrypted)──→ Gateway
Gateway decrypts request (TLS termination)
Gateway ──HTTP or internal HTTPS──→ Backend Service

Why:
  - Centralize certificate management
  - One place handles TLS handshake overhead
  - Backend services don't each need public certs

Internal HTTP:
  - Simpler, less overhead inside private network
  - Acceptable when internal network is trusted

Internal HTTPS:
  - Stronger security (zero-trust)
  - Better for cloud/multi-tenant/regulated systems
```

### What Is Middleware?

Logic that runs **before or after** your core handler. Cross-cutting concerns.

```
Common middleware:
  - Authentication / authorization
  - Logging / request ID injection
  - Rate limiting
  - CORS
  - Compression
  - Body size limits
  - Input validation
  - Timeout handling
  - Panic / exception recovery
  - Metrics collection
  - CSRF protection
  - IP filtering
```

### Full Request Path with All Concepts

```
1. Client sends HTTPS request
2. API Gateway receives it
3. Gateway does:
   - SSL termination (decrypt)
   - Auth check
   - Gateway rate limit
   - Routing
4. Backend service receives request
5. Service middleware does:
   - Logging, tracing
   - Server-side/business rate limit
   - Validation
6. Service checks cache (Redis)
7. If cache miss, queries DB
8. Returns response back through gateway to browser
```

### One-Sentence Definitions

```
Token bucket    → Allow bursts, control long-term request rate
Client limiter  → Polite client behavior, NOT real security
Distributed     → Shared quota across many server instances
Cache server    → Fast shared in-memory store for hot data
API gateway     → Front door that handles cross-cutting concerns
Middleware      → Reusable logic around request processing
Gateway limiter → Edge-level protection (DDoS, abuse)
Server limiter  → Business-logic-level protection (quotas)
```
