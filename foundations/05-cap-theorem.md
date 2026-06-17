# 05 — CAP Theorem

## The Scenario That Causes the Problem

You have a Replication setup — 1 Primary, 3 Replicas, possibly spread across different servers or data centers.

Now imagine the network connection between the Primary and one Replica breaks — not a server crash, just the network link between them being cut off. The Replica is still running fine, just isolated.

```
Primary ----X (network broken)----  Replica 2
   |
   ✓-- Replica 1 (still connected)
   ✓-- Replica 3 (still connected)
```

> This situation — where parts of a system can't communicate due to network failure — is called a **Network Partition**.

---

## The Core Decision

A user's request lands on the isolated Replica 2, which now holds potentially outdated data (it can't sync with Primary anymore). Replica 2 has two choices:

**Choice 1 — Refuse to respond (or error out) until it can confirm the data is correct**
> *"I can't guarantee this is up-to-date. I'd rather say nothing than give wrong data."*
> Prioritizes **Consistency**.

**Choice 2 — Respond anyway with whatever data it has, even if possibly stale**
> *"I'll give you what I have, even though I can't confirm it's the latest. Something is better than nothing."*
> Prioritizes **Availability**.

---

## CAP Theorem — In Plain Words

| Letter | Meaning |
|---|---|
| **C — Consistency** | Every read gets the most recent write; no stale data, ever |
| **A — Availability** | Every request gets *some* response; the system never refuses to answer |
| **P — Partition Tolerance** | The system keeps operating despite network partitions |

> When a network partition happens (and in real distributed systems, it eventually will), you can only pick **Consistency OR Availability** — not both, at that moment.

**Important nuance:** Partition Tolerance is essentially assumed as a given in any real distributed system — networks aren't perfectly reliable. The actual real-world choice being made is between **C and A**, specifically *during* a partition. If there's no partition (everything healthy and connected), a system can be both consistent and available simultaneously. CAP only forces a tradeoff when something goes wrong.

---

## Applying CAP to Real Systems

| System | Priority during partition | What it sacrifices |
|---|---|---|
| Likes, view counts, social media counters | **Availability (AP)** | Consistency — may show slightly stale data |
| Bank balance, stock prices, payment systems | **Consistency (CP)** | Availability — may refuse to respond rather than show wrong data |

This is the same reasoning already used in [03-databases.md](./03-databases.md) (Replication Lag) and [04-caching.md](./04-caching.md) (TTL-based staleness) — CAP Theorem is the formal, unifying name for that recurring tradeoff.

---

## Why SQL Databases Typically Lean CP

Traditional SQL databases (MySQL, PostgreSQL, Oracle) are commonly set up with **synchronous replication** — when a write happens, the Primary waits for replica confirmation before reporting success.

```
Write Request → Primary writes → Primary waits for Replica to confirm → THEN responds "success"
```

> If a partition prevents that confirmation, many SQL setups will **block or fail the write** rather than proceed without it — choosing Consistency over Availability.

SQL databases also enforce **ACID transactions** — strict correctness guarantees, even at the cost of speed or availability.

---

## Why NoSQL Databases Typically Lean AP

Many NoSQL databases (Cassandra, DynamoDB-style systems) use **asynchronous replication** by default — a write succeeds on one node immediately, and other nodes catch up in the background.

```
Write Request → Node writes → Immediately responds "success" → Syncs to other nodes in the background
```

> During a partition, the node serving a read doesn't wait to confirm it's fully in sync — it just responds with whatever data it currently has, choosing Availability over Consistency.

### Quorum-Based Writes (the tunable mechanism)

Rather than requiring **all** replicas to confirm (strict SQL-style), many NoSQL systems let you configure **how many replicas must agree** before a write is considered successful:

```
3 replicas total
Quorum = 2  →  only 2 of 3 must confirm → faster, more available, slightly less consistent
Quorum = 3  →  all 3 must confirm → slower, less available, fully consistent
```

> This is a tunable knob in many NoSQL systems — consistency vs availability isn't always a hard rule, but a configurable default tendency based on the database's architecture.

---

## The Precise Summary

> It's not that "SQL = CP" and "NoSQL = AP" by strict definition. It's that **SQL databases are typically architected with synchronous replication and strict consistency by default**, while **many NoSQL databases are architected with asynchronous replication and tunable consistency, defaulting toward availability**.

---

## Key Takeaways

- A Network Partition is a communication breakdown between parts of a distributed system, not a server crash
- During a partition, a system must choose between Consistency (refuse/delay rather than be wrong) and Availability (always respond, even if stale)
- Partition Tolerance is assumed; the real tradeoff in practice is C vs A
- The CP/AP choice should match the system's actual needs — social features lean AP, financial/critical systems lean CP
- SQL's synchronous replication + ACID transactions → naturally leans CP
- NoSQL's asynchronous replication + tunable quorum → naturally leans AP
- Quorum settings allow NoSQL systems to shift along the C/A spectrum rather than being fixed

---

## System Design Relevance

| Concept | Where it appears in system design |
|---|---|
| CAP Theorem | The theoretical foundation behind every consistency/availability decision in a distributed system |
| Network Partition | Multi-region deployments, cross-data-center replication |
| AP Systems | Social feeds, like/view counters, recommendation systems |
| CP Systems | Payments, banking, inventory systems, booking systems (seat/ticket reservations) |
| Quorum | Distributed databases (Cassandra, DynamoDB) tuning consistency vs availability per use case |

---

## Phase 1 — Foundations: Complete ✅

This concludes Phase 1. Topics covered, in order:
1. DNS and Networking
2. Load Balancers
3. Databases (SQL vs NoSQL, Indexing, Replication, Sharding)
4. Caching
5. CAP Theorem

**Next — Phase 2: Designing Simple Systems**, starting with the URL Shortener.
