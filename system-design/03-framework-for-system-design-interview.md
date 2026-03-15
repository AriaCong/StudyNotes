# A Framework for System Design Interviews

A **structured 4-step framework** to navigate any system design interview. The goal is to demonstrate **clear thinking**, not to memorize specific designs. Communication and trade-off analysis matter more than the "right" answer.

---

## Table of Contents

1. [Overview: The 4-Step Framework](#overview-the-4-step-framework)
2. [Step 1: Understand the Problem & Establish Scope](#step-1-understand-the-problem--establish-scope)
3. [Step 2: Propose High-Level Design & Get Buy-In](#step-2-propose-high-level-design--get-buy-in)
4. [Step 3: Design Deep Dive](#step-3-design-deep-dive)
5. [Step 4: Wrap Up](#step-4-wrap-up)
6. [Time Allocation](#time-allocation)
7. [Red Flags vs Green Flags](#red-flags-vs-green-flags)

---

## Overview: The 4-Step Framework

```
┌──────────────────────────────────────────────────────────────────────┐
│                    45-Minute Interview Flow                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1              Step 2              Step 3           Step 4     │
│  Understand &        High-Level          Deep             Wrap       │
│  Scope               Design              Dive             Up        │
│                                                                      │
│  ┌─────────┐        ┌─────────┐        ┌──────────┐    ┌────────┐  │
│  │ Ask     │───────>│ Draw    │───────>│ Zoom     │───>│Summary │  │
│  │Questions│        │Blueprint│        │Into Key  │    │& Trade │  │
│  │& Clarify│        │& Align  │        │Components│    │-offs   │  │
│  └─────────┘        └─────────┘        └──────────┘    └────────┘  │
│                                                                      │
│  3-10 min            10-15 min          10-25 min       3-5 min     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

> **Key insight:** This is a **collaborative design session**, not a test. Think of the interviewer as a teammate.

---

## Step 1: Understand the Problem & Establish Scope

### Why This Step Matters

Jumping into design without clarifying requirements is the **#1 mistake**. Even experienced engineers fail when they solve the wrong problem.

```
DON'T:  "Design a news feed"  →  immediately draw boxes
DO:     "Design a news feed"  →  ask 10+ clarifying questions first
```

### What to Ask

```
┌──────────────────────────────────────────────────────────────────┐
│                    Clarifying Questions Map                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Functional Requirements          Non-Functional Requirements    │
│  ─────────────────────            ───────────────────────────    │
│  • What features to build?        • How many users? (scale)     │
│  • Who are the users?             • What's acceptable latency?  │
│  • What are the key use cases?    • Availability target?        │
│  • What inputs/outputs?           • Consistency model?          │
│  • What is out of scope?          • Read-heavy or write-heavy?  │
│                                   • Data retention? Growth?     │
│  Constraints & Context                                           │
│  ────────────────────                                            │
│  • Existing tech stack?                                          │
│  • Mobile + web or just one?                                     │
│  • Can we use 3rd-party services?                                │
│  • What's the DAU / traffic pattern?                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Example: "Design a Chat System"

```
Good clarifying questions:

Q: 1:1 or group chat or both?                → Scope
Q: Mobile app, web, or both?                  → Clients
Q: What scale? How many DAU?                  → Scale
Q: Is there a message size limit?             → Constraints
Q: Do we need end-to-end encryption?          → Security
Q: How long is chat history stored?           → Storage
Q: Do we need read receipts, online status?   → Features
Q: Push notifications when offline?           → Features
```

### Output of Step 1

```
Write down agreed-upon:
  ✓ Functional requirements (2-3 key features)
  ✓ Non-functional requirements (scale, latency, availability)
  ✓ Back-of-envelope estimates (QPS, storage, bandwidth)
  ✓ What's explicitly OUT of scope
```

---

## Step 2: Propose High-Level Design & Get Buy-In

### Goal

Draw a **blueprint** that covers all key components. Get the interviewer to agree before going deeper.

### Approach

```
1. Sketch the API design (key endpoints)
   ↓
2. Draw high-level diagram (clients → LB → services → storage)
   ↓
3. Walk through core use cases on the diagram
   ↓
4. Ask: "Does this look reasonable? Anything you'd change?"
```

### API Design First

Define the contract before drawing boxes.

```
Example: URL Shortener

POST /api/v1/shorten
  Request:  { "long_url": "https://..." }
  Response: { "short_url": "https://tinyurl.com/abc123" }

GET /api/v1/{short_url}
  Response: HTTP 301 redirect to long_url
```

### High-Level Diagram Template

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Client  │────>│ Load Balancer│────>│  Web Servers  │
│(Mobile/  │     │              │     │  (Stateless)  │
│ Web)     │     └──────────────┘     └──────┬───────┘
└──────────┘                                  │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                              ┌────────┐ ┌────────┐ ┌────────┐
                              │ Cache  │ │  DB    │ │ Queue  │
                              │(Redis) │ │(SQL/   │ │(Async) │
                              │        │ │ NoSQL) │ │        │
                              └────────┘ └────────┘ └────────┘
```

### Walk Through Use Cases

For each core feature, trace the **request flow** through the diagram:

```
"When a user posts a tweet..."

  1. Client sends POST /tweets to Load Balancer
  2. LB routes to web server
  3. Web server validates, writes to DB
  4. Fanout service pushes to followers' caches
  5. Followers' home timelines update
```

> **Tip:** The interviewer may challenge or redirect. That's good — it means you're having a design conversation, not giving a lecture.

---

## Step 3: Design Deep Dive

### Goal

Zoom into **2-3 components** the interviewer cares most about. This is where you demonstrate depth.

### How to Pick What to Deep Dive

```
Priority:
  1. Whatever the interviewer asks about     ← always follow their lead
  2. The most technically challenging part    ← shows depth
  3. The part that differentiates the system  ← shows understanding

Typical deep dive areas:
  • Database schema & access patterns
  • Caching strategy & invalidation
  • Scaling bottlenecks & solutions
  • Data partitioning / sharding strategy
  • Consistency vs availability trade-offs
  • API rate limiting & abuse prevention
  • Failure modes & recovery
```

### Deep Dive Flow

```
┌─────────────────────────────────────────────────────────────┐
│                   Deep Dive Structure                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. State the problem clearly                               │
│     "The bottleneck here is that reads are 100x writes..."  │
│                                                             │
│  2. Propose 2-3 approaches with trade-offs                  │
│     Option A: Cache-aside with TTL (simple, eventual)       │
│     Option B: Write-through cache (consistent, more writes) │
│     Option C: CQRS with event sourcing (complex, scalable)  │
│                                                             │
│  3. Pick one and justify                                    │
│     "Given our scale and consistency needs, I'd go with A   │
│      because..."                                            │
│                                                             │
│  4. Detail the implementation                               │
│     Schema, algorithms, failure handling                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Example Deep Dives by System

```
┌─────────────────┬──────────────────────────────────────────────┐
│ System          │ Likely Deep Dive Topics                      │
├─────────────────┼──────────────────────────────────────────────┤
│ URL Shortener   │ Hash function, collision handling, 301 vs 302│
│ News Feed       │ Fanout on write vs read, ranking algorithm   │
│ Chat System     │ WebSocket vs polling, message ordering       │
│ Rate Limiter    │ Algorithm choice, distributed coordination   │
│ Web Crawler     │ URL frontier, politeness, dedup, trap detect │
│ Notification    │ Reliability, retry, dedup, priority queuing  │
│ Search          │ Inverted index, ranking, relevance scoring   │
└─────────────────┴──────────────────────────────────────────────┘
```

---

## Step 4: Wrap Up

### What to Cover (3-5 minutes)

```
1. Recap the design
   └─ 2-3 sentence summary of how the system works

2. Identify bottlenecks & improvements
   └─ "If we had more time, I'd address..."
   └─ Shows self-awareness and forward thinking

3. Error handling & edge cases
   └─ What happens when X fails?
   └─ Monitoring, alerting, logging

4. Scaling to next level
   └─ "To go from 1M to 10M users, we'd need to..."
   └─ Sharding, multi-region, CDN, etc.

5. Operational concerns
   └─ Metrics, dashboards, deployment strategy
```

### Don't Do This in Wrap Up

```
✗ Repeat everything you already said
✗ Introduce entirely new components
✗ Say "I think it's perfect" — always acknowledge trade-offs
```

---

## Time Allocation

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   Step 1: Scope        ████████░░░░░░░░░░░░░░░░░░  3-10 min │
│                                                              │
│   Step 2: High-Level   ░░░░░░░░████████████░░░░░░░ 10-15 min│
│                                                              │
│   Step 3: Deep Dive    ░░░░░░░░░░░░░░░░░░░████████ 10-25 min│
│                                                              │
│   Step 4: Wrap Up      ░░░░░░░░░░░░░░░░░░░░░░░████  3-5 min │
│                                                              │
│   Total: ~45 minutes                                         │
│                                                              │
│   ▓▓▓ = where most candidates spend too little time          │
│   Step 1 is usually rushed. Step 3 is where you win.         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Red Flags vs Green Flags

### Red Flags (Instant Negative Signals)

```
✗ Jump straight into solution without asking questions
✗ Design in silence without explaining thought process
✗ Refuse to take hints or push back on interviewer's suggestions
✗ Obsess over one component, ignore the full picture
✗ Over-engineer from the start (sharding for 1K users)
✗ Give up when stuck instead of reasoning through alternatives
✗ Ignore non-functional requirements (scale, reliability)
✗ No trade-off discussion — everything is "the best"
```

### Green Flags (What Interviewers Love)

```
✓ Ask great clarifying questions before designing
✓ Communicate clearly — narrate your thought process
✓ Start simple, then add complexity only when justified
✓ Discuss trade-offs for every major decision
✓ Use back-of-envelope math to justify choices
✓ Adapt when the interviewer steers the conversation
✓ Acknowledge what you don't know
✓ Consider failure modes and edge cases proactively
✓ Mention monitoring, observability, and operability
```

### The Meta-Framework

```
For EVERY design decision, follow this pattern:

  ┌──────────────────────────────────────────────┐
  │  1. What are the options?                     │
  │  2. What are the trade-offs of each?          │
  │  3. Which do I pick and WHY?                  │
  │  4. What are the risks of my choice?          │
  └──────────────────────────────────────────────┘

This pattern — applied consistently — is what separates
a "pass" from a "strong pass" in system design interviews.
```
