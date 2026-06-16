# 03 — Databases

## SQL vs NoSQL

### The Core Difference

| | **SQL** | **NoSQL** |
|---|---|---|
| Structure | Strict schema — tables, columns, relationships | Flexible schema — documents, key-value, etc. |
| Data shape | Every row **must** follow the same structure | Each document can have different fields |
| Relationships | Built using foreign keys (e.g. `userId`, `photoId`) | Often **embedded** within the document itself |
| Example | MySQL, PostgreSQL, Oracle | MongoDB, DynamoDB, Cassandra |

### Terminology Mapping

```
SQL world          →  NoSQL world
-----------------------------------
Table              →  Collection
Row                →  Document
Column             →  Field
```

---

## Designing the "Likes" Feature — A Case Study

### The Naive Approach (Wrong)

A `likeYN` flag per user per photo would require a row for **every user-photo combination**, even if no like happened. At Instagram's scale (2B users × 500M photos) — this is an impossible number of rows.

### The Correct Approach

> A row only gets created **when the like actually happens.**

```
likes table (source of truth)
| userId | photoId | created_at |
|--------|---------|------------|
| 123    | 456     | 2026-06-10 |
| 789    | 456     | 2026-06-11 |
```

### The Read Performance Problem

Running `SELECT COUNT(*) FROM likes WHERE photoId = 456` on every feed view (with millions of rows per popular photo) is far too expensive.

### The Fix — Denormalization

Keep a separate, fast-to-read count alongside the detailed table:

```
photos table (denormalized for speed)
| photoId | userId | imageUrl | likeCount |
|---------|--------|----------|-----------|
| 456     | 123    | url...   | 2300000   |
```

- `likes` table → answers *"who liked this"*, *"did I like this"*
- `likeCount` on `photos` → answers *"how many likes"* fast, without counting rows

**Tradeoff:** Faster reads, but now two places must be kept in sync (risk of inconsistency if one write fails and the other succeeds).

---

## Why Not Embed Likes Directly in the Photo Document (NoSQL)?

```json
{
  "photoId": 456,
  "likedBy": ["123", "789", ... 2.3 million ids]
}
```

**Problem:** Most NoSQL databases have a document size limit (MongoDB = 16MB per document). An unbounded array inside a single document will eventually break it.

**Key Insight:**
> The problem is never "how much total data exists" — databases handle millions of rows/documents fine. The problem is **one single row/document growing unbounded.**

**Correct NoSQL design** — same idea as SQL, just renamed:

```json
// likes collection — many small, independent documents
{ "userId": 123, "photoId": 456 }
{ "userId": 789, "photoId": 456 }
```

---

## When to Choose SQL vs NoSQL

The deciding factor isn't "which can achieve the result" (both can) — it's **how the data evolves at scale**.

| Factor | SQL | NoSQL |
|---|---|---|
| Schema changes at small scale | Easy | Easy |
| Schema changes at **massive scale** (billions of rows) | Risky — `ALTER TABLE` can lock/slow a live table for minutes to hours | Instant — new documents can simply have new fields |
| Data consistency guarantees | Strong — enforced by the database | Weak — enforced by application code |

### Applied Example

| System | Choice | Why |
|---|---|---|
| **Likes** | SQL | Fixed structure, never changes, relationship-based |
| **User Profiles** | NoSQL | Fields evolve over time (bio, links, custom fields), billions of users, avoids risky schema migrations |

> Real systems often use **both** — this is called **Polyglot Persistence**: using different databases for different parts of the same system, based on what each part needs.

---

## Indexing

### The Problem — Full Table Scan

Without optimization, a query like:
```sql
SELECT * FROM likes WHERE userId = 123 AND photoId = 456;
```
checks every row one by one — O(n). For billions of rows, this is extremely slow.

### The Fix — Index

> An index is a separate, sorted data structure that lets the database jump directly to relevant rows, instead of scanning everything — O(log n) instead of O(n).

**Analogy:** Like using a textbook's index page to jump to "Database... page 245" instead of reading the entire book page by page.

### The Tradeoff

| | Reads (SELECT) | Writes (INSERT/UPDATE/DELETE) |
|---|---|---|
| **With Index** | Much faster | Slower — every index must also be updated |
| **Without Index** | Slow (full scan) | Faster — nothing extra to update |

> Index the columns your actual queries filter on — not every column blindly.

### Composite Index

A single index built on **multiple columns together**, sorted hierarchically:

```sql
CREATE INDEX idx_user_photo ON likes(userId, photoId);
```

**Mental model — nested map, not nested filter:**
```java
Map<Integer, Map<Integer, Row>> index;
// index.get(123).get(456) → direct jump, one operation
```

**Analogy:** A book index sorted by chapter name first, then page number within each chapter.

**Key limitation:** A composite index `(userId, photoId)` only helps if the query filters by `userId` (the first column). A query filtering **only** by `photoId` cannot use this index efficiently — same as trying to find "page 245" in an index sorted by chapter name only.

**Practical takeaway:** For the `likes` table, both `userId` and `photoId` need their own index (or a composite, depending on query patterns) because different queries filter by different columns:
- `WHERE userId = X AND photoId = Y` → benefits from `idx_userId` (and composite even more)
- `WHERE photoId = Y` alone → needs `idx_photoId` separately; `idx_userId` doesn't help here at all

---

## Replication

### The Problem

A single database server handling all reads and writes has two issues:
1. **Performance** — limited capacity, will slow down or crash under heavy load
2. **Single Point of Failure (SPOF)** — if it crashes, the entire app goes down and data may be lost

### The Fix — Replication

> Keep multiple copies of the same database, kept in sync with each other.

```
                Writes go here
                      ↓
                 Primary DB
                /     |     \
         (syncs data to all replicas)
              /       |       \
        Replica 1  Replica 2  Replica 3
              ↑          ↑         ↑
                   Reads go here
```

**The rule (Primary-Replica / Master-Slave pattern):**
- **All writes** → go to Primary only
- **All reads** → spread across Replicas
- Primary continuously syncs changes to Replicas

**Why writes can't go to multiple replicas randomly:** If two replicas accept writes independently without real-time coordination, conflicting/inconsistent data can result. A single Primary avoids this by being the one source of truth.

### Replication Lag

> Replicas don't update instantly — there's a small delay while data syncs from Primary to Replicas.

**Scenario:** You like a photo (write → Primary) → refresh immediately (read → Replica 2, not yet synced) → you don't see your own like for a moment.

This is called **Eventual Consistency** — the data will become correct eventually, but may be momentarily stale.

**Severity depends on the system:**
| System | Replication Lag Impact |
|---|---|
| Likes | Mildly annoying — self-corrects in milliseconds/seconds, no real damage |
| Stock prices, bank balances | Seriously dangerous — stale data can cause real financial loss or incorrect decisions |

> This ties directly into **CAP Theorem** (covered next): some systems favor Availability (always respond, even if slightly stale — e.g. likes), others favor Consistency (must be exact, even if it means waiting — e.g. bank balances).

---

## Sharding

### The Problem

Replication solves read scaling, but **all writes still go to one Primary**. Eventually, a single Primary can't handle the write volume or data size alone — even with a bigger machine (vertical scaling has limits).

### The Fix — Sharding (Horizontal Scaling for Databases)

> Split data across multiple independent database servers (shards), where each shard holds only a portion of the total data.

```
Shard 1 → Users 1 - 1,000,000
Shard 2 → Users 1,000,001 - 2,000,000
Shard 3 → Users 2,000,001 - 3,000,000
Shard 4 → Users 3,000,001 - 4,000,000
```

Each shard is independent — own CPU, RAM, disk — and typically has its **own replication** set up:

```
Shard 1 (Users 1-1M)
   ├── Primary
   ├── Replica 1
   └── Replica 2
```

> Sharding and Replication work together, not as alternatives: Sharding spreads data/writes across servers; Replication protects each shard against SPOF and spreads its reads.

### Sharding Strategies

**1. Range-Based Sharding**
Split by a range of values (e.g., user ID ranges).

**Problem — Hotspotting:** If older/more active users cluster in one range (e.g., Shard 1 has the earliest, most active users), that shard gets overloaded with writes while others sit idle. Replication doesn't fix this — replicas only help reads, not writes, and the Primary of the hot shard is still overwhelmed.

**2. Hash-Based Sharding**
```
shardNumber = hash(userId) % numberOfShards
```
Scrambles users pseudo-randomly across shards, so active and inactive users get evenly mixed — no single shard becomes a hotspot.

### Tradeoff: Range-Based vs Hash-Based

| | Range-Based | Hash-Based |
|---|---|---|
| Even load distribution | ❌ Hotspots possible | ✅ Evenly spread |
| Range queries (e.g., "all users from Jan 2020") | ✅ Easy — single shard | ❌ Hard — must query all shards and combine results |

---

## Key Takeaways

- A NoSQL document = a SQL row; avoid unbounded growth in a single row/document
- SQL = strict schema, ideal for fixed, relational data
- NoSQL = flexible schema, ideal for evolving, varied data
- Real systems often use **polyglot persistence** — different databases for different needs
- Indexes trade write speed for read speed; index based on actual query patterns
- Composite indexes only help when the query uses the leading column(s)
- Replication solves both SPOF and read scaling; writes go to Primary, reads spread across Replicas
- Replication lag causes eventual consistency — acceptable for low-stakes data, dangerous for high-stakes data
- Sharding solves write/data scaling by splitting data across servers
- Range-based sharding risks hotspots; hash-based sharding distributes evenly but loses easy range queries

---

## System Design Relevance

| Concept | Where it appears in system design |
|---|---|
| SQL vs NoSQL | Every system's data layer decision |
| Denormalization | Feed systems, counters, dashboards |
| Indexing | Any system with frequent lookups at scale |
| Replication | High availability, read-heavy systems |
| Eventual Consistency | Social media features (likes, views, follower counts) |
| Sharding | Systems with massive write volume or data size (Instagram, Uber, WhatsApp) |
| CAP Theorem (next topic) | The theoretical foundation behind consistency vs availability tradeoffs |
