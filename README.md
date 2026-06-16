# System Design Learning

A structured, hands-on journey through system design — from fundamentals to designing real-world systems like Twitter, WhatsApp, and YouTube.

---

## Approach

- Learn by doing — every concept is tied to a real problem
- Document thought process, not just answers
- Focus on **tradeoffs** — there is no single correct answer in system design
- Every design follows a fixed framework (see below)

---

## Design Framework

Every system design problem will follow this structure:

1. **Clarify Requirements** — what exactly are we building?
2. **Estimate Scale** — how many users, how much data?
3. **Design the API** — what endpoints exist?
4. **Design the Data Model** — what does the database look like?
5. **High Level Design** — draw the components
6. **Deep Dive** — zoom into the hard parts
7. **Identify Bottlenecks** — what breaks at scale?

---

## Learning Path

### Phase 1 — Foundations
Building blocks every system is made of.

| # | Topic | Status |
|---|---|---|
| 01 | DNS and Networking | ✅ Done |
| 02 | Load Balancers | ✅ Done |
| 03 | Databases (SQL vs NoSQL, Indexing, Replication, Sharding) | ✅ Done |
| 04 | Caching | 🔄 Up next |
| 05 | CAP Theorem | ⏳ Pending |

### Phase 2 — Simple Systems
| # | System | Status |
|---|---|---|
| 01 | URL Shortener | ⏳ Pending |
| 02 | Pastebin | ⏳ Pending |
| 03 | Rate Limiter | ⏳ Pending |

### Phase 3 — Medium Systems
| # | System | Status |
|---|---|---|
| 01 | WhatsApp / Chat | ⏳ Pending |
| 02 | Instagram / Image Sharing | ⏳ Pending |
| 03 | Notification System | ⏳ Pending |

### Phase 4 — Complex Systems
| # | System | Status |
|---|---|---|
| 01 | Twitter Feed / News Feed | ⏳ Pending |
| 02 | YouTube | ⏳ Pending |
| 03 | Uber / Location-based | ⏳ Pending |

---

## Repository Structure

```
system-design-learning/
├── README.md
├── foundations/
│   ├── 01-dns-and-networking.md
│   ├── 02-load-balancers.md
│   ├── 03-databases.md
│   ├── 04-caching.md
│   └── 05-cap-theorem.md
├── designs/
│   ├── 01-url-shortener/
│   ├── 02-pastebin/
│   ├── 03-rate-limiter/
│   ├── 04-whatsapp/
│   ├── 05-instagram/
│   ├── 06-twitter-feed/
│   └── 07-youtube/
└── diagrams/
```

---

## Key Principle

> System design is always about **tradeoffs**. The goal is not to find the perfect answer — it is to make informed decisions and explain your reasoning clearly.
