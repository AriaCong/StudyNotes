# System Design: Scaling from 100 to 1 Million Users

A progressive guide to evolving a web application architecture from a single server to handling millions of users. Each stage addresses the **current bottleneck** with the **minimum necessary complexity**.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [Database Selection](#1-database-selection)
3. [Scaling Strategy](#2-scaling-strategy)
4. [Caching Architecture](#3-caching-architecture)
5. [Session Management & Auth](#4-session-management--auth)
6. [Async Processing & Message Queues](#5-async-processing--message-queues)
7. [Production Deployment Patterns](#6-production-deployment-patterns)
8. [CDN & Content Delivery](#7-cdn--content-delivery)
9. [The Scaling Roadmap](#8-the-scaling-roadmap)
10. [Common Mistakes & Senior Thinking](#9-common-mistakes--senior-thinking)

---

## Core Principle

> **Architecture evolves by removing the current bottleneck.**

```
1. Keep it simple early
2. Add observability
3. Split tiers (app vs DB)
4. Add caching + async
5. Scale reads
6. Scale writes
7. Partition data + teams
```

---

## 1. Database Selection

### Relational (Postgres/MySQL) — Use When:

- **Strong consistency + transactions** (money, inventory, bookings)
- **Ad-hoc queries**, reporting, flexible filters, and **joins**
- **Constraints** (foreign keys, unique) to enforce data correctness
- One "source of truth" that many services rely on

### NoSQL — Use When:

| Type | Examples | Best For |
|------|----------|----------|
| **Key-Value** | Redis, DynamoDB | Sessions, carts, feature flags, rate limits |
| **Document** | MongoDB | Flexible/evolving schemas, nested objects |
| **Column Store** | Cassandra, Bigtable | Write-heavy, time-series, logs, events |
| **Graph** | Neo4j | Relationship traversals, social graphs |

### Decision Summary

| Tool | Good For | Dangerous When |
|------|----------|----------------|
| Postgres | Business logic, integrity, ad-hoc queries | Extreme horizontal scale without planning |
| DynamoDB | Massive auto-scale + predictable key-based queries | Ad-hoc queries, complex joins |
| Redis | Speed, caching, counters, pub/sub | Used as primary DB without persistence |

### Redis Deep Dive

```
In-memory | Single-threaded | Microsecond latency | Optional persistence
```

**Use for:** Cache, session store, leaderboards, rate limiting, distributed locks, pub/sub

**Do NOT use when:** Strong durability needed, complex relational queries, multi-row transactions, data must never be lost

**If Redis crashes:**
- As cache: Fine - cache rebuilds, temporarily slower responses
- As primary store: **Data loss possible** - this is why Redis should not be your only data store

---

## 2. Scaling Strategy

### Vertical vs Horizontal

```
Vertical Scaling                    Horizontal Scaling
┌──────────────┐                    ┌────────────────────────────────┐
│              │                    │         Load Balancer          │
│  Bigger      │                    └──┬─────────┬─────────┬────────┘
│  Machine     │                       │         │         │
│  (CPU/RAM)   │                    ┌──┴──┐   ┌──┴──┐   ┌──┴──┐
│              │                    │App 1│   │App 2│   │App 3│
└──────────────┘                    └──┬──┘   └──┬──┘   └──┬──┘
                                       └─────────┼─────────┘
                                              ┌──┴──┐
                                              │ DB  │
                                              └─────┘
```

**Start vertical when:** early-stage, moderate traffic, simplicity matters

**Go horizontal on app tier when:** traffic spikes, need HA, can run stateless instances

### DB Scaling Path (in order)

```
1. Vertical scale DB
   ↓
2. Add indexes + query tuning
   ↓
3. Caching layer (Redis)
   ↓
4. Read replicas (scale reads)
   ↓
5. Partition / shard (scale writes) ← last resort, complexity jumps
```

---

## 3. Caching Architecture

### The Cache Layer Stack

```
┌─────────────────────────────┐
│    Browser HTTP Cache        │  ← per-user, automatic via headers
│    (Cache-Control, ETag)     │
├─────────────────────────────┤
│    CDN / Edge Cache          │  ← shared, global, infrastructure
│    (Cloudflare, Akamai)      │
├─────────────────────────────┤
│    Frontend App Cache        │  ← per-user, in JS memory
│    (React Query / SWR)       │
├─────────────────────────────┤
│    Backend Distributed Cache │  ← shared across instances
│    (Redis / Memcached)       │
├─────────────────────────────┤
│    In-Process Cache          │  ← per-instance, not shared
│    (local map / LRU)         │
├─────────────────────────────┤
│    Database                  │  ← source of truth
└─────────────────────────────┘
```

Each layer reduces load on the layer below it.

### Request Flow Through Cache Layers

```
User clicks page
      │
      ▼
Browser checks HTTP cache ──── HIT ──→ Return cached response
      │ MISS
      ▼
CDN checks edge cache ──────── HIT ──→ Return cached response
      │ MISS
      ▼
App checks React Query ─────── HIT ──→ Return cached data
      │ MISS
      ▼
Backend checks Redis ────────── HIT ──→ Return cached data
      │ MISS
      ▼
Database query ──────────────────────→ Return + populate caches
```

### Cache Strategies

#### 1. Cache-Aside (Lazy Loading) — Most Common

```
App ──→ Redis ──→ HIT? Return
              └──→ MISS? ──→ DB ──→ Store in Redis ──→ Return
```

Best for: user profiles, product pages, dashboards

#### 2. Write-Through

```
App ──→ Write DB ──→ Write Redis immediately
```

Best for: read-heavy systems, strong consistency desired

#### 3. Write-Behind (Write-Back)

```
App ──→ Write Redis ──→ Async flush to DB
```

Best for: logging, analytics counters (dangerous if cache crashes before flush)

#### 4. Read-Through

```
App ──→ Cache (cache auto-loads from DB on miss)
```

### Cache Invalidation Methods

| Method | How It Works | Use Case |
|--------|-------------|----------|
| **TTL** | Auto-expire after N seconds | General purpose |
| **Write-through** | Update DB, then delete/update cache | Strong consistency |
| **Event-driven** | DB update → publish event → subscribers invalidate | Microservices |
| **Cache-aside delete** | On write, delete cache key; next read re-populates | Most common pattern |

### When to Introduce a Cache

- DB CPU > 70% consistently
- Same query executed thousands of times
- p95 latency increasing
- Expensive computations repeated

### Eviction Policies

| Policy | Use When |
|--------|----------|
| **LRU** (Least Recently Used) | Recency-based access (browsing, dashboards) |
| **LFU** (Least Frequently Used) | Frequency-based (trending content, popular products) |
| **TTL-based** | Fixed expiry (auth tokens, rate limits) |

### Hot Key Problem

When one cache key gets 100k+ requests/sec → Redis CPU spike → bottleneck

**Mitigations:** Replication, sharding, in-process cache for that key, CDN, precompute & distribute

---

## 4. Session Management & Auth

### Stateless Web Tier

Any request can go to any app instance. Session state is stored externally, not in instance memory.

### Auth Patterns Compared

```
JWT Bearer Token                     Session Cookie + Store
┌──────────────┐                     ┌──────────────┐
│   Browser    │                     │   Browser    │
│  stores JWT  │                     │  stores      │
│  in cookie   │                     │  session_id  │
│  or header   │                     │  cookie      │
└──────┬───────┘                     └──────┬───────┘
       │ Authorization: Bearer <jwt>        │ Cookie: session_id=abc
       ▼                                    ▼
┌──────────────┐                     ┌──────────────┐
│   Server     │                     │   Server     │
│  verifies    │                     │  looks up    │
│  signature   │                     │  session in  │
│  (no lookup) │                     │  Redis/DB    │
└──────────────┘                     └──────┬───────┘
                                            ▼
                                     ┌──────────────┐
                                     │    Redis     │
                                     │  session:abc │
                                     │  → user_id   │
                                     └──────────────┘
```

| Dimension | JWT Bearer | Session (cookie + store) |
|-----------|-----------|--------------------------|
| Client sends | `Authorization: Bearer <jwt>` | `Cookie: session_id=...` |
| Contains user info | Yes (claims in token) | No (just opaque id) |
| Server needs shared store | No (basic) | Yes (Redis/DB) |
| Logout effect | Not immediate (until expiry) | Immediate (delete session) |
| Revocation | Extra work (blocklist) | Built-in (remove session) |
| Best for | APIs, multi-service, mobile | Browser apps, SSR, strong control |

### Practical Default: httpOnly Cookie + Redis

```
Browser                          Next.js BFF                    Go API                Redis
   │                                 │                            │                     │
   │ POST /login (credentials)       │                            │                     │
   │────────────────────────────────>│                            │                     │
   │                                 │ forward to Go API          │                     │
   │                                 │───────────────────────────>│                     │
   │                                 │                            │ verify credentials   │
   │                                 │                            │ create session       │
   │                                 │                            │────────────────────>│
   │                                 │                            │ session:abc→user:42  │
   │                                 │        {session_id: abc}   │                     │
   │                                 │<───────────────────────────│                     │
   │ Set-Cookie: session_id=abc;     │                            │                     │
   │ HttpOnly; Secure; SameSite=Lax  │                            │                     │
   │<────────────────────────────────│                            │                     │
   │                                 │                            │                     │
   │ GET /dashboard (cookie auto)    │                            │                     │
   │────────────────────────────────>│                            │                     │
   │                                 │ X-Session-Id: abc          │                     │
   │                                 │───────────────────────────>│                     │
   │                                 │                            │ validate in Redis    │
   │                                 │                            │────────────────────>│
   │                                 │                            │ valid, user_id=42    │
   │                                 │        {user data}         │<────────────────────│
   │                                 │<───────────────────────────│                     │
   │         rendered HTML           │                            │                     │
   │<────────────────────────────────│                            │                     │
```

---

## 5. Async Processing & Message Queues

### Why Async?

Move heavy work off the critical request path. User gets fast response; processing happens in background.

### Photo Upload & Processing Flow

```
┌─────────┐     ┌──────────────┐     ┌──────────┐     ┌───────────┐     ┌────────────┐
│ Browser │────>│ Next.js BFF  │────>│  Go API  │────>│   Queue   │────>│  Workers   │
└─────────┘     └──────────────┘     └────┬─────┘     │(SQS/Rabbit│     └─────┬──────┘
                                          │           └───────────┘           │
                                          │                                   │
                                     ┌────┴──────┐                      ┌─────┴──────┐
                                     │ Postgres  │                      │  S3/GCS    │
                                     │(metadata) │                      │ (images)   │
                                     └───────────┘                      └────────────┘
```

### Detailed Request Lanes

```
Browser        Next.js(BFF)      Go API           Queue          Worker         S3/GCS        Postgres
  │                │                │                │              │              │              │
  │ POST /photos   │                │                │              │              │              │
  │───────────────>│  forward       │                │              │              │              │
  │                │───────────────>│ auth user      │              │              │              │
  │                │                │ create row     │              │              │              │
  │                │                │──────────────────────────────────────────────────────────>│
  │                │                │ gen presign URL│              │              │              │
  │                │  {url, photoId}│                │              │              │              │
  │<───────────────│<───────────────│                │              │              │              │
  │                │                │                │              │              │              │
  │ PUT file ──────────────────────────────────────────────────────────────────>│              │
  │ (direct to S3) │                │                │              │              │              │
  │                │                │                │              │              │              │
  │ POST /complete │                │                │              │              │              │
  │───────────────>│───────────────>│ status=UPLOADED│              │              │              │
  │                │                │───────────────>│ publish msg  │              │              │
  │                │  202 Accepted  │                │              │              │              │
  │<───────────────│<───────────────│                │              │              │              │
  │                │                │                │              │              │              │
  │                │                │                │  consume msg │              │              │
  │                │                │                │─────────────>│              │              │
  │                │                │                │              │ get original │              │
  │                │                │                │              │─────────────>│              │
  │                │                │                │              │ process      │              │
  │                │                │                │              │ upload thumbs│              │
  │                │                │                │              │─────────────>│              │
  │                │                │                │              │ update status│              │
  │                │                │                │              │──────────────────────────>│
  │                │                │                │              │ ACK msg      │              │
  │                │                │                │<─────────────│              │              │
  │                │                │                │              │              │              │
  │ GET /photos/id │                │                │              │              │              │
  │───────────────>│───────────────>│ status=PROCESSED              │              │              │
  │  {thumbnails}  │                │                │              │              │              │
  │<───────────────│<───────────────│                │              │              │              │
```

### Queue Message Design

```json
{
  "photoId": "123",
  "originalKey": "uploads/123/original.jpg",
  "ops": ["thumb_256", "web_1280", "strip_exif"]
}
```

**Never put image bytes in the queue.** Store in object storage, pass references.

### Worker Scaling

- Autoscale based on: queue depth, oldest message age, CPU
- Workers are idempotent (safe to retry)
- Dead-letter queue (DLQ) for poison messages
- Status tracking: `UPLOADING → UPLOADED → PROCESSING → PROCESSED / FAILED`

---

## 6. Production Deployment Patterns

### BFF Pattern (Backend-for-Frontend) — Recommended

Browser only talks to Next.js. Go API stays private.

```
                          ┌────────────────────────────────────────┐
                          │              Cloudflare                 │
                          │    DNS + TLS + WAF + CDN               │
                          └───────────────┬────────────────────────┘
                                          │  https://app.example.com
                                          ▼
                                 ┌──────────────────┐
                                 │   Public ALB      │
                                 └────────┬─────────┘
                                          ▼
                          ┌───────────────────────────────┐
                          │  Next.js BFF (Node runtime)    │
                          │  - SSR pages                    │
                          │  - /api/* routes (BFF proxy)    │
                          │  - Auth/session handling         │
                          └───────────────┬───────────────┘
                                          │  PRIVATE (VPC only)
                                          ▼
                                 ┌──────────────────┐
                                 │  Internal ALB     │
                                 └────────┬─────────┘
                                          ▼
                          ┌───────────────────────────────┐
                          │  Go API (business logic)       │
                          └───────────────┬───────────────┘
                                          │
                            ┌─────────────┴──────────────┐
                            ▼                            ▼
                   ┌────────────────┐          ┌────────────────┐
                   │ Redis          │          │ Postgres        │
                   │ (sessions/     │          │ (source of      │
                   │  cache)        │          │  truth)         │
                   └────────────────┘          └────────────────┘
```

**Why BFF?**
- Go API not publicly reachable → smaller attack surface
- No CORS headaches
- Auth centralized at BFF
- Can evolve API without breaking external consumers

### AWS Infrastructure Mapping

| Component | AWS Service |
|-----------|------------|
| CDN/WAF | CloudFront or Cloudflare in front |
| Public LB | Application Load Balancer (internet-facing) |
| Internal LB | ALB (private subnets) |
| Next.js | ECS Fargate (or Vercel) |
| Go API | ECS Fargate (private subnets) |
| Database | RDS Postgres |
| Cache/Sessions | ElastiCache Redis |
| Object Storage | S3 |
| Queue | SQS |

### Kubernetes Equivalent

```
Browser (Internet)
   │
   │ https://app.example.com
   ▼
┌────────────────────────────┐
│ K8s Ingress (public)        │
└───────────────┬────────────┘
                ▼
┌────────────────────────────┐
│ Deployment: nextjs-bff      │
│ Service: ClusterIP          │
└───────────────┬────────────┘
                │  cluster-internal DNS
                ▼
┌────────────────────────────┐
│ Service: go-api (ClusterIP) │  ← NOT exposed externally
└───────────────┬────────────┘
                ▼
┌────────────────────────────┐
│ Deployment: go-api          │
└───────────────┬────────────┘
        ┌───────┴────────┐
        ▼                ▼
   Redis (svc)     Postgres (managed)
```

---

## 7. CDN & Content Delivery

### CDN Is Infrastructure, Not Code

```
User ──→ DNS resolves app.example.com ──→ CDN Edge ──→ Origin Server
                                              │
                                         Cache HIT?
                                        /          \
                                      Yes           No
                                       │             │
                                  Return cached   Forward to origin
                                  response        Cache response
```

Frontend does NOT "call CDN". It calls your domain. CDN intercepts via DNS routing.

### What to Cache Where

| Layer | What | Headers / Config |
|-------|------|-----------------|
| **Static assets** (JS/CSS/images) | Long TTL, versioned URLs | `Cache-Control: public, max-age=31536000, immutable` |
| **Public SSR HTML** | Short TTL at CDN | `Cache-Control: public, s-maxage=60, stale-while-revalidate=300` |
| **Public API GET** | Short TTL at CDN | `Cache-Control: public, s-maxage=30` |
| **Private/auth pages** | Never cache at CDN | `Cache-Control: private, no-store` |

### CDN Invalidation

| Method | When to Use |
|--------|------------|
| **TTL-based** | Dynamic content; accept staleness within window |
| **Versioned URLs** | Static assets; new deploy = new hash = new URL |
| **API purge** | Specific content updated; purge exact URLs |
| **Cache tags** | Group-based purge (if CDN supports it) |

### CloudFront vs Cloudflare

| Aspect | CloudFront (AWS) | Cloudflare |
|--------|-----------------|------------|
| Ecosystem | AWS-only, deep S3/ALB integration | Cloud-agnostic, works with any origin |
| DNS | Separate (Route 53) | Acts as authoritative DNS |
| WAF | AWS WAF (separate config) | Built-in, easy to configure |
| Edge compute | Lambda@Edge | Workers |

---

## 8. The Scaling Roadmap

### Stage 0: ~1K Users — Single Server MVP

```
┌────────────────────────┐
│       Single VM         │
│  ┌──────┐  ┌────────┐  │
│  │ App  │  │Postgres│  │
│  │(Node/│  │        │  │
│  │ Go)  │  │        │  │
│  └──────┘  └────────┘  │
└────────────────────────┘
```

**Focus:** Correct data model, basic indexes, ship fast.

**Red flag:** microservices, Kafka, multi-region, sharding at this stage.

---

### Stage 1: ~10K Users — 2-Tier + Managed Services

```
┌──────────────┐
│ Load Balancer │
└──────┬───────┘
   ┌───┴───┐
┌──┴──┐ ┌──┴──┐       ┌───────────┐
│App 1│ │App 2│       │Managed DB  │
│     │ │     │──────>│(RDS/Cloud  │
└─────┘ └─────┘       │ SQL)       │
                       └───────────┘
+ CDN for static content
+ S3 for file uploads
+ Centralized logs + metrics + traces
```

**Trigger:** deploys and spikes hurt; DB and app compete for resources.

---

### Stage 2: ~100K Users — Caching + Background Jobs

```
┌──────────────┐
│ Load Balancer │
└──────┬───────┘
   ┌───┴───┐
┌──┴──┐ ┌──┴──┐     ┌───────┐     ┌──────────┐
│App 1│ │App 2│────>│ Redis │     │  Queue   │
└──┬──┘ └──┬──┘     │(cache)│     │(SQS/     │
   └───┬───┘        └───┬───┘     │RabbitMQ) │
       │                │         └────┬─────┘
   ┌───┴────────────────┴───┐         │
   │      Postgres           │    ┌────┴─────┐
   └─────────────────────────┘    │ Workers  │
                                  └──────────┘
```

**Add Redis cache-aside** for hot reads (profiles, feeds, config).

**Add async processing** for email, image processing, notifications.

---

### Stage 3: ~1M Users — Read Scaling + Search

```
                    ┌────────────┐
                    │   Writes   │
                    └─────┬──────┘
                          ▼
                    ┌────────────┐
              ┌────>│  Primary   │
              │     │  Postgres  │
              │     └─────┬──────┘
              │      replication
         ┌────┴───┐  ┌───┴────┐  ┌────────────────┐
         │Replica 1│  │Replica2│  │ Elasticsearch  │
         │ (reads) │  │(reads) │  │ (search/filter)│
         └─────────┘  └────────┘  └────────────────┘
```

**Scale reads** with Postgres replicas. **Add search** (Elasticsearch). Keep Postgres as source of truth.

---

### Stage 4: ~3-5M Users — Data Partitioning + Domain Separation

**Before sharding:**
- Correct indexes, remove N+1 queries
- Partition tables by time (events/logs)
- Materialized views for heavy reads
- Aggressive caching + async writes

**Then split by domain:**

```
┌────────────────┐  ┌──────────────────┐  ┌────────────────┐
│ Core DB         │  │ Events/Analytics  │  │ Search Index   │
│ (users, orders, │  │ (clickstream,     │  │ (denormalized  │
│  payments)      │  │  logs, metrics)   │  │  for queries)  │
└────────────────┘  └──────────────────┘  └────────────────┘
```

---

### Stage 5: ~10M Users — Multi-Region + Selective NoSQL

```
        Region A                              Region B
┌─────────────────────┐              ┌─────────────────────┐
│  CDN + App Servers   │              │  CDN + App Servers   │
│  Redis (local)       │              │  Redis (local)       │
│  DB Replica          │              │  DB Replica          │
└──────────┬──────────┘              └──────────┬──────────┘
           └──────────────┬───────────────────────┘
                          ▼
                   Primary DB (Region A)

+ DynamoDB/Cassandra for specific access patterns
  (sessions, feeds, counters, id→doc lookups)
+ Sharding only when truly necessary
  (by tenant, geography, or userId hash)
```

---

### TL;DR Scaling Path

```
Single server ──→ App/DB split + LB ──→ Redis + queue workers
       │                  │                       │
      ship             stability              performance
                                                  │
                                                  ▼
              Read replicas + search ──→ Partition + precompute
                       │                          │
                   read scale                  DB relief
                                                  │
                                                  ▼
                            Selective NoSQL + multi-region ──→ Sharding
                                       │                          │
                                 global scale               final boss
```

---

## 9. Common Mistakes & Senior Thinking

### Mistakes Mid-Level Engineers Make

1. **Start with microservices** — trades shipping speed for complexity too early
2. **Skip observability** — guessing bottlenecks leads to wrong fixes
3. **Treat cache like a database** — Redis as source of truth → crash = outage
4. **Over/under-index randomly** — without measuring query plans
5. **N+1 queries** — hidden at small scale, destroys latency at 1M
6. **No backpressure** — system collapses instead of degrading gracefully
7. **Ignore data growth** — works at 10GB, nightmare at 2TB

### What Senior Engineers Optimize For

1. **Simplicity per stage** — minimum architecture that meets reliability + performance
2. **Clear scaling levers** — know which knob to turn:
   - add app instances → add cache → add replicas → partition → async jobs
3. **Reliability + degradation** — timeouts, circuit breakers, retries with jitter
4. **Data correctness boundaries** — strongly consistent (payments) vs eventual (feeds)
5. **Cost awareness** — bandwidth + storage + read amplification at scale
6. **Operability** — backups, migrations, incident response, dashboards
7. **Team boundaries** — microservices as org scaling tool, not technical flex

### Architecture Evolution Mental Model

| Early | Later |
|-------|-------|
| Everything synchronous | Critical path minimal; everything else async |
| DB answers every question | DB is truth; caches/search/derived stores serve reads |
| One schema | Transactional + denormalized read models + event streams |
| One deployment | Multiple services *only when teams and scaling demand it* |
