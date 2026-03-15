# Back-of-the-Envelope Estimation

Quick calculations to evaluate design alternatives using **thought experiments** and **common performance numbers**. The goal is not precision — it's to determine if a design is **feasible** before diving into details.

---

## Table of Contents

1. [Power of 2](#power-of-2)
2. [Latency Numbers Every Programmer Should Know](#latency-numbers)
3. [Availability & SLA](#availability--sla)
4. [Estimation Framework](#estimation-framework)
5. [Common Calculations](#common-calculations)
6. [Example: Twitter QPS & Storage (10M DAU)](#example-twitter-qps--storage-10m-dau)
7. [Example: Media Extension (Photos + Video)](#example-media-extension-photos--video)
8. [Tips & Common Mistakes](#tips--common-mistakes)

---

## Power of 2

Data volume units expressed as powers of 2. Memorize these — they're the building blocks.

| Power | Approx Value | Name | Short |
|-------|-------------|------|-------|
| 10 | 1 Thousand | 1 Kilobyte | 1 KB |
| 20 | 1 Million | 1 Megabyte | 1 MB |
| 30 | 1 Billion | 1 Gigabyte | 1 GB |
| 40 | 1 Trillion | 1 Terabyte | 1 TB |
| 50 | 1 Quadrillion | 1 Petabyte | 1 PB |

### Quick Conversion Rules

```
1 byte = 8 bits
1 ASCII char = 1 byte
1 Unicode char (UTF-8) = ~4 bytes max
1 int = 4 bytes (32-bit) or 8 bytes (64-bit)
1 UUID = 16 bytes (128 bits)
1 typical tweet = ~250 bytes (140 chars + metadata)
1 photo (compressed) = ~200 KB - 1 MB
1 short video clip = ~5-50 MB
```

---

## Latency Numbers

These are **order-of-magnitude** reference points (originally from Jeff Dean, ~2020 values).

```
Operation                                   Time            Scale
─────────────────────────────────────────────────────────────────────
L1 cache reference                          1 ns            ████
L2 cache reference                          4 ns            ████
Branch mispredict                           3 ns            ████
Mutex lock/unlock                           17 ns           ████
Main memory reference                       100 ns          ████████
Compress 1KB (Snappy)                       10 μs           ████████████
Read 1 MB from memory                       3 μs            ████████████
SSD random read                             16 μs           ████████████████
Read 1 MB sequentially from SSD             49 μs           ████████████████████
Round trip in same datacenter               500 μs          ████████████████████████
Read 1 MB sequentially from disk            825 μs          ████████████████████████████
Disk seek                                   2 ms            ████████████████████████████████
Read 1 MB from network                      10 ms           ████████████████████████████████████
Send packet CA → NL → CA                    150 ms          ████████████████████████████████████████
```

### Key Takeaways

```
                Memory           SSD             Disk           Network
               ┌──────┐      ┌──────┐        ┌──────┐       ┌──────┐
Speed:         │ ~ns  │  >>  │ ~μs  │   >>   │ ~ms  │  >>   │ ~ms  │
               └──────┘      └──────┘        └──────┘       └──────┘
               fastest                                       slowest
```

1. **Memory is fast, disk is slow** — avoid disk seeks when possible
2. **Compression before sending** — saves network time (network is slowest)
3. **Sequential reads >> Random reads** — both on SSD and disk
4. **Inter-datacenter round trip is expensive** — ~150ms cross-continent
5. **Simple compression is fast** — Snappy at 10 μs is almost free vs network cost

---

## Availability & SLA

### The Nines Table

| Availability % | Downtime / Year | Downtime / Day | Common Name |
|----------------|----------------|----------------|-------------|
| 99% | 3.65 days | 14.4 min | Two nines |
| 99.9% | 8.77 hours | 1.44 min | Three nines |
| 99.99% | 52.6 min | 8.6 sec | Four nines |
| 99.999% | 5.26 min | 0.86 sec | Five nines |

Most cloud services target **99.9% - 99.99%**.

### SLA Math — Serial vs Parallel

```
Serial (both must work):        Parallel (either works):

  A ──→ B                         ┌── A ──┐
                                  │       │──→
                                  └── B ──┘

  Availability = A × B            Availability = 1 - (1-A)(1-B)

  Example:                        Example:
  99.9% × 99.9% = 99.8%          1 - (0.001)(0.001) = 99.9999%
  (worse than either alone)       (better than either alone)
```

> **Rule of thumb:** Every serial dependency degrades availability. Redundancy improves it.

---

## Estimation Framework

### Step-by-Step Approach

```
1. State assumptions clearly
   ↓
2. Round numbers for easy math (use powers of 10)
   ↓
3. Label units (KB, QPS, MB/s) — prevents errors
   ↓
4. Work through calculation step by step
   ↓
5. Sanity check: does the result make sense?
```

### Common Conversion Constants

```
1 day    = 86,400 sec  ≈ 100,000 sec  (10^5)  ← use this for quick math
1 month  ≈ 2.5 million sec             (2.5 × 10^6)
1 year   ≈ 30 million sec              (3 × 10^7)
```

### QPS (Queries Per Second) Formula

```
QPS = Daily Active Users × Queries per User / Seconds per Day

        DAU × avg_queries
QPS = ─────────────────────
           86,400

Peak QPS ≈ 2× to 10× average QPS (consumer apps often use 10×)
```

### Storage Formula

```
Storage = Users × Data per User × Retention Period

Example:
  100M users × 1 KB profile × 1 = 100 GB (just profiles)
  100M users × 500 KB media/day × 365 days × 5 years = 9.1 PB
```

### Bandwidth Formula

```
Bandwidth = QPS × Average Response Size

Example:
  10K QPS × 50 KB avg response = 500 MB/s = 4 Gbps
```

---

## Common Calculations

### Web Server Capacity

```
1 modern web server ≈ handles 1K-10K concurrent connections
1 application server ≈ 200-600 QPS (with DB calls, mixed workload)
1 server (CPU-heavy) ≈ 50-100 QPS
```

### Database Throughput (rough)

```
┌──────────────────────────────────────────────────┐
│              Throughput Estimates                  │
├──────────────┬──────────┬────────────────────────┤
│ System       │ Reads/s  │ Writes/s               │
├──────────────┼──────────┼────────────────────────┤
│ MySQL/PG     │ ~10K     │ ~5K (single node)      │
│ Redis        │ ~100K    │ ~100K                  │
│ Cassandra    │ ~10K     │ ~10K (per node, linear)│
│ DynamoDB     │ configurable, auto-scales         │
│ Elasticsearch│ ~5K      │ ~5K (per node)         │
└──────────────┴──────────┴────────────────────────┘
```

### Object Sizes (Common)

```
┌─────────────────────────────────────────┐
│ What                     │ Size         │
├──────────────────────────┼──────────────┤
│ User profile row         │ ~1 KB        │
│ Tweet / short post       │ ~250 bytes   │
│ Like / follow edge       │ ~64 bytes    │
│ Metadata per object      │ ~500 bytes   │
│ Compressed photo (web)   │ 200 KB-1 MB  │
│ Avatar / thumbnail       │ 10-50 KB     │
│ Short video (1 min)      │ 5-50 MB      │
│ Log line                 │ ~200 bytes   │
│ URL (short)              │ ~100 bytes   │
└──────────────────────────┴──────────────┘
```

---

## Example: Twitter QPS & Storage (10M DAU)

### Assumptions

```
Users:
  - DAU: 10,000,000

Actions per user/day:
  - Timeline reads: 30
  - Other reads (tweet detail/profile/search): 20
  - Likes (write): 5.3
  - Tweets posted (write): 0.2  (≈ 1 tweet per 5 days)
  - Follows (write): 0.1

Traffic shape:
  - Peak factor: 10× average (common for consumer apps)

Data sizes (per write):
  - Tweet row (text + metadata): 1.5 KiB raw
  - Like edge: 64 B raw
  - Follow edge: 64 B raw

Overhead & replication:
  - Tweets: 2.0× (indexes, extra columns)
  - Likes: 1.5×
  - Follows: 1.5×
  - Replication factor: 3×
  - Retention: 365 days

Server capacity:
  - 1 stateless API instance: ~600 QPS (mixed workload)
  - Headroom: 1.5× (deploys + node loss + variance)
```

### 1) QPS Estimate

```
Per user/day:
  Reads  = 30 + 20 = 50
  Writes = 5.3 + 0.2 + 0.1 = 5.6
  Total  = 55.6

Total requests/day:
  10M × 55.6 = 556,000,000 req/day

Average QPS:
  556M / 86,400 ≈ 6,435 QPS

Peak QPS (10×):
  ≈ 64,352 QPS
```

### Read vs Write Split

```
Reads/day:  10M × 50   = 500M  →  Avg 5,787 QPS  →  Peak 57,870
Writes/day: 10M × 5.6  = 56M   →  Avg 648 QPS    →  Peak 6,481
```

### 2) Storage Estimate

```
Writes per day:
  Tweets:  10M × 0.2 = 2,000,000
  Likes:   10M × 5.3 = 53,000,000
  Follows: 10M × 0.1 = 1,000,000

Daily storage (with overhead, before replication):
  Tweets:  2M × 1.5 KiB × 2.0   = 6.0 GiB/day
  Likes:   53M × 64 B × 1.5     = 4.74 GiB/day
  Follows: 1M × 64 B × 1.5      = 0.09 GiB/day
  Total: ~10.55 GiB/day

Yearly (365 days × 3× replication):
  10.55 × 365 × 3 ≈ 11,552 GiB ≈ 11.3 TiB/year
```

> This excludes media (images/videos). Adding media usually dwarfs everything else.

### 3) Cache Sizing

```
Timeline reads/day: 10M × 30 = 300M
  Avg timeline QPS: 300M / 86,400 ≈ 3,472
  Peak timeline QPS (10×): 34,722

Cache model: per-user timeline response, TTL = 30 seconds
  Response payload: ~6 KiB + 1 KiB overhead = ~7 KiB per entry
  Entries at peak: 34,722 × 30 ≈ 1,041,660
  Cache needed: 1,041,660 × 7 KiB ≈ 6.8 GiB

With headroom (2× for evictions, fragmentation):
  Timeline cache: ~14 GiB

Practical total cache tier (+ user profiles, auth, rate limits, etc.):
  ~64 GiB
```

### 4) Number of API Servers

```
Peak QPS: 64,352
Per-instance capacity: 600 QPS
Base instances: 64,352 / 600 ≈ 108
With 1.5× headroom: 108 × 1.5 = 162 instances
Across 3 AZs: ~54 per AZ
```

### Summary Table

```
┌─────────────────────────────────────────────────────────┐
│ Metric                    │ Value                        │
├───────────────────────────┼──────────────────────────────┤
│ Avg QPS                   │ ~6.4K                        │
│ Peak QPS                  │ ~64K                         │
│ Peak Reads / Writes       │ ~57.9K / ~6.5K               │
│ Storage/day (no repl)     │ ~10.6 GiB                    │
│ Storage/year (3× repl)    │ ~11.3 TiB                    │
│ Timeline cache            │ ~14 GiB                      │
│ Total cache tier          │ ~64 GiB                      │
│ API servers               │ ~162 instances (600 QPS each) │
└───────────────────────────┴──────────────────────────────┘
```

---

## Example: Media Extension (Photos + Video)

Extends the same Twitter-like 10M DAU scenario.

### Media Assumptions

```
Media mix (of 2M tweets/day):
  - 35% include 1 photo → 700,000 photos/day
  - 5% include 1 video  → 100,000 videos/day

Stored sizes (after processing, multiple renditions):
  - Photo per upload: ~1 MB (original + 2-3 resized variants)
  - Video per upload: ~30 MB (HLS segments + multiple bitrates + thumbs)

Upload sizes (raw from client):
  - Photo: ~2 MB
  - Video: ~20 MB

Client delivery per user/day:
  - Timeline media: ~280 KB per timeline view
  - Full image opens: 3 × 500 KB = 1.5 MB
  - Video watches: 0.1 × 5 MB = 0.5 MB

Replication: 3×
Retention: 365 days
CDN hit ratio: 95%
```

### 1) Upload QPS

```
Total uploads/day: 700K + 100K = 800K
Avg upload QPS: 800K / 86,400 ≈ 9.3
Peak upload QPS (10×): ~93

Upload QPS is small. The big cost is storage + bandwidth + transcoding.
```

### 2) Object Storage Growth

```
Daily stored media (logical, before replication):
  Photos: 700K × 1 MB   = 700 GB/day
  Videos: 100K × 30 MB   = 3,000 GB/day
  Total: ~3,700 GB/day  ≈ 3.5 TB/day

Yearly (365 days, before replication):
  3.5 TB × 365 ≈ 1,288 TB ≈ 1.29 PB/year

With 3× replication:
  ≈ 3.86 PB/year
```

### 3) CDN Egress + Bandwidth

```
A) Timeline media:
   300M views/day × 280 KB ≈ 78.2 TiB/day

B) Full image views:
   10M × 3 × 500 KB ≈ 14.0 TiB/day

C) Video streams:
   10M × 0.1 × 5 MB ≈ 4.77 TiB/day

Total CDN egress: ~97 TiB/day

Avg bandwidth: 97 TiB / 86,400 ≈ 9.9 Gbps
Peak bandwidth (10×): ~99 Gbps
```

### 4) Origin Bandwidth (CDN miss = 5%)

```
Origin egress/day: 97 TiB × 5% ≈ 4.85 TiB/day
Avg origin bandwidth: ~0.49 Gbps
Peak: ~4.9 Gbps

This is why CDNs are non-optional at this scale.
```

### 5) Upload Ingress

```
Photos: 700K × 2 MB  = 1.4 TB/day
Videos: 100K × 20 MB = 2.0 TB/day
Total: ~3.24 TiB/day

Avg ingress: ~0.33 Gbps
Peak: ~3.3 Gbps
```

### 6) Media Processing Fleet

```
CPU-seconds per item:
  Photo processing: 0.2 CPU-sec/photo
  Video transcode:  60 CPU-sec/video

Compute total/day:
  Photos: 700K × 0.2 = 140K CPU-sec ≈ 1.6 CPU-days
  Videos: 100K × 60  = 6M CPU-sec   ≈ 69 CPU-days

With 2× headroom for peaks/backlog:
  ~150 CPU cores for transcoding
```

### Media Summary Table

```
┌─────────────────────────────────────────────────────────┐
│ Metric                    │ Value                        │
├───────────────────────────┼──────────────────────────────┤
│ Media uploads/day         │ 700K photos, 100K videos     │
│ Upload QPS                │ avg ~9.3, peak ~93           │
│ Storage growth            │ ~3.5 TB/day → ~3.9 PB/yr    │
│ CDN egress                │ ~97 TiB/day                  │
│ Bandwidth                 │ avg ~9.9 Gbps, peak ~99 Gbps│
│ Origin egress (95% hit)   │ ~4.85 TiB/day               │
│ Upload ingress            │ ~3.24 TiB/day               │
│ Processing fleet          │ ~100-200 CPU cores           │
│ API servers               │ ~162 (API) + ~4-8 (upload)  │
└───────────────────────────┴──────────────────────────────┘
```

---

## Tips & Common Mistakes

### Do

- **Round aggressively** — use powers of 10 (`86,400 ≈ 10^5`)
- **Write down assumptions** — interviewer wants to see your reasoning
- **Label units** — QPS, MB/s, GB, etc. — prevents order-of-magnitude errors
- **Back-calculate to sanity check** — does the result feel right?
- **Know your reference numbers** — latency table, power of 2, QPS ranges
- **Separate reads vs writes** — most systems are read-heavy (10:1 to 100:1)

### Don't

- Don't aim for exact numbers — the goal is **order of magnitude**
- Don't forget **peak vs average** — systems must handle peaks (use 10× for consumer apps)
- Don't ignore **read vs write ratio** — determines caching and replica strategy
- Don't confuse **storage vs bandwidth** — one is cumulative, other is rate
- Don't skip **compression** in bandwidth estimates — real systems compress
- Don't forget **replication factor** — production data is replicated 3×
- Don't forget **overhead** — indexes, metadata, fragmentation add 1.5-2×

### Cheat Sheet: Quick Multipliers

```
                    ×1K        ×1M         ×1B
1 KB per user   →  1 MB    →  1 GB     →  1 TB
1 MB per user   →  1 GB    →  1 TB     →  1 PB
1 request/sec   →  86K/day →  2.6M/mo  →  31M/yr
```
