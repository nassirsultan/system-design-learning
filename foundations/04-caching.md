# 04 — Caching

## The Problem

Even after denormalizing `likeCount` onto the `photos` table (see [03-databases.md](./03-databases.md)), every photo view still triggers a database read:

```sql
SELECT likeCount FROM photos WHERE photoId = 456;
```

At millions of reads per second, even a simple, indexed query adds up in cost and latency.

---

## Does Data Need to Be Unchanging to Be Cached?

No. The deciding factor isn't whether data changes — it's the **relationship between how often it's read vs how often it's written**, and whether slight staleness is acceptable.

> Caching works best when **read frequency ≫ write frequency**, and the data doesn't need to be perfectly accurate at every instant.

`likeCount` fits this perfectly:
- Read millions of times (every photo view, by anyone)
- Written rarely, relatively (only on a like/unlike action)
- Being off by a few likes for a few seconds is harmless

---

## What is a Cache?

> A cache is a fast, temporary storage layer — usually in-memory (RAM) — that holds frequently accessed data, avoiding repeated trips to the slower database.

```
Without Cache:
User views photo → Query Database → Get likeCount → Slow (disk read, every time)

With Cache:
User views photo → Check Cache (in-memory) → Get likeCount → Fast (microseconds)
```

### Why Cache (RAM) is Faster Than Database (Disk)

Rough scale, just to anchor the magnitude:

```
RAM access      → ~100 nanoseconds
SSD access      → ~100 microseconds   (~1,000x slower than RAM)
Database query  → 1-10 milliseconds   (network + parsing + disk I/O combined)
```

> A cache hit can be roughly **10,000x to 100,000x faster** than a full database query.

**Redis** is the most common in-memory data store used for this purpose.

---

## Cache-Aside Pattern (Lazy Loading)

The most common caching flow:

```
                          ┌─────────┐
User Request → Spring Boot → Redis  (check cache first)
                          └─────────┘
                               ↓ (if not found in cache)
                          ┌─────────┐
                          │Database │
                          └─────────┘
```

**Step by step:**
1. User requests `likeCount` for photo 456
2. App checks Redis first
3. **Cache Hit** → Redis returns it instantly, database is never touched
4. **Cache Miss** → App queries the database, stores the result in Redis for next time, then returns it to the user

---

## TTL (Time To Live) in Caching

Same underlying concept as DNS TTL, applied to cached data:

```
SET likeCount:456 2300000 EX 60
```
> Stores `likeCount` for photo 456, and expires it automatically after 60 seconds.

**Timeline example:**
```
t=0s   → Cache Miss → DB queried → likeCount=2300000 stored in Redis, TTL=60s
t=5s   → Someone likes the photo → DB updated to 2300001 → Redis still says 2300000
t=10s  → User views photo → Redis hit → shows 2300000 (slightly stale, but fast)
t=60s  → TTL expires → Redis automatically deletes the cached value
t=61s  → User views photo → Cache Miss → DB queried again → fresh value cached again
```

> The window between the DB update and the TTL expiry is the same phenomenon covered in [03-databases.md](./03-databases.md) under Replication Lag — **Eventual Consistency**. The same concept appears here in a completely different context.

---

## Cache Invalidation Strategies

Instead of just waiting for TTL to expire, the cache can be actively kept fresh.

| Strategy | How it works | Tradeoff |
|---|---|---|
| **TTL-only** | Cache naturally expires after a set time; next read repopulates it | Simple, but data can be stale for the entire TTL window |
| **Write-Through** | Every database write also immediately updates the cache | Cache is never stale, but every write now costs more (two writes instead of one) |

### Choosing Between Them

| Data Type | Strategy | Why |
|---|---|---|
| `likeCount` | TTL-based (Cache-Aside) | Staleness for a few seconds is harmless; simpler to implement |
| Stock price / Bank balance | Write-Through | Must always be accurate; worth the extra write cost |

> This mirrors the same Consistency vs Availability tradeoff seen in Replication — covered formally next in CAP Theorem.

---

## Cache Eviction

Cache memory (RAM) is limited — unlike disk, it can't grow indefinitely. When the cache is full and a new item needs to be cached, an existing item must be removed.

### Eviction Policies

| Policy | Removes | Use case |
|---|---|---|
| **LRU** (Least Recently Used) | Item not accessed in the longest time | General purpose — most common default |
| **LFU** (Least Frequently Used) | Item accessed the fewest total times | When access *frequency* matters more than *recency* |
| **FIFO** (First In First Out) | Oldest item added, regardless of usage | Simple, rarely optimal |
| **TTL-based eviction** | Items past their expiration time | Time-sensitive data |

**LRU example:**
```
Cache (max 3 items): [Photo 456] [Photo 789] [Photo 123]

New request: Photo 999 needs caching, but cache is full
LRU checks: which photo was accessed LEAST recently? → Photo 789
→ Remove Photo 789, add Photo 999

Cache now: [Photo 456] [Photo 999] [Photo 123]
```

> Redis allows configuring the eviction policy via `maxmemory-policy`. **LRU is the default and most common choice** for general-purpose caching.

---

## Key Takeaways

- Cache best fits read-heavy, write-light data where slight staleness is acceptable
- RAM-based caches (Redis) are ~10,000x+ faster than database queries
- Cache-Aside (Lazy Loading): check cache → miss → query DB → populate cache → return
- TTL controls automatic expiration, same concept as DNS TTL
- Cache Invalidation: TTL-only (simple, eventual consistency) vs Write-Through (always fresh, costlier writes)
- Cache Eviction policies (when memory is full): LRU is the most common default

---

## System Design Relevance

| Concept | Where it appears in system design |
|---|---|
| Cache-Aside | Most read-heavy systems (feeds, profiles, counters) |
| TTL | Session data, rate limiting, temporary tokens |
| Write-Through | Financial data, inventory counts, anything consistency-critical |
| LRU Eviction | Any memory-bound cache (Redis, CDN edge caches, browser cache) |
| Eventual Consistency (via TTL) | Social media counters, view counts, like counts |
