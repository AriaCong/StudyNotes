# Back-of-the-Envelope Estimation

Quick calculations to evaluate design alternatives using **thought experiments** and **common performance numbers**. The goal is not precision — it's to determine if a design is **feasible** before diving into details.

---

## Table of Contents

1. [Power of 2](#power-of-2)
2. [Latency Numbers Every Programmer Should Know](#latency-numbers)
3. [Availability & SLA](#availability--sla)
4. [Estimation Framework](#estimation-framework)
5. [Common Calculations](#common-calculations)
6. [Example: Twitter QPS & Storage](#example-twitter-qps--storage)
7. [Tips & Common Mistakes](#tips--common-mistakes)

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

Peak QPS ≈ 2× to 5× average QPS (use 2× or 3× as rule of thumb)
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
1 application server ≈ 200-500 QPS (with DB calls)
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
│ Metadata per object      │ ~500 bytes   │
│ Compressed photo (web)   │ 200 KB-1 MB  │
│ Avatar / thumbnail       │ 10-50 KB     │
│ Short video (1 min)      │ 5-50 MB      │
│ Log line                 │ ~200 bytes   │
│ URL (short)              │ ~100 bytes   │
└──────────────────────────┴──────────────┘
```

---

## Example: Twitter QPS & Storage

### Assumptions

```
- 300M monthly active users (MAU)
- 50% use daily → 150M DAU
- Each user posts 2 tweets/day on average
- 10% of tweets contain media
- Data retained for 5 years
```

### QPS Estimate

```
Write QPS:
  150M × 2 tweets / 86,400 sec ≈ 150M × 2 / 10^5
                                ≈ 3,000 tweets/sec (avg)
  Peak write QPS ≈ 3,000 × 3 = ~9,000/sec

Read QPS (home timeline):
  150M users × 10 reads/day / 86,400 ≈ ~17,000 QPS
  Peak ≈ 50,000 QPS
```

### Storage Estimate

```
Text storage (per day):
  150M × 2 tweets × 250 bytes = 75 GB/day

Media storage (per day):
  150M × 2 × 10% × 500 KB = 15 TB/day

5-year storage:
  Text: 75 GB × 365 × 5 ≈ 137 TB
  Media: 15 TB × 365 × 5 ≈ 27 PB
```

### Bandwidth Estimate

```
Outgoing (reads dominate):
  17K QPS × 50 KB (avg tweet + metadata) ≈ 850 MB/s ≈ 6.8 Gbps
  (peak: ~20 Gbps)
```

### Estimation Summary

```
┌─────────────────────────────────────────────────┐
│ Metric              │ Average    │ Peak          │
├─────────────────────┼────────────┼───────────────┤
│ Write QPS           │ ~3K/s      │ ~9K/s         │
│ Read QPS            │ ~17K/s     │ ~50K/s        │
│ Text storage/day    │ 75 GB      │               │
│ Media storage/day   │ 15 TB      │               │
│ 5-year total        │ ~27 PB     │               │
│ Outgoing bandwidth  │ 850 MB/s   │ ~2.5 GB/s     │
└─────────────────────┴────────────┴───────────────┘
```

---

## Tips & Common Mistakes

### Do

- **Round aggressively** — use powers of 10 (`86,400 ≈ 10^5`)
- **Write down assumptions** — interviewer wants to see your reasoning
- **Label units** — QPS, MB/s, GB, etc. — prevents order-of-magnitude errors
- **Back-calculate to sanity check** — does the result feel right?
- **Know your reference numbers** — latency table, power of 2, QPS ranges

### Don't

- Don't aim for exact numbers — the goal is **order of magnitude**
- Don't forget **peak vs average** — systems must handle peaks
- Don't ignore **read vs write ratio** — most systems are read-heavy (10:1 to 100:1)
- Don't confuse **storage vs bandwidth** — one is cumulative, other is rate
- Don't skip **compression** in bandwidth estimates — real systems compress

### Cheat Sheet: Quick Multipliers

```
                    ×1K        ×1M         ×1B
1 KB per user   →  1 MB    →  1 GB     →  1 TB
1 MB per user   →  1 GB    →  1 TB     →  1 PB
1 request/sec   →  86K/day →  2.6M/mo  →  31M/yr
```
