# 01 — DNS and Networking

## What is DNS?

DNS stands for **Domain Name System**. It translates human-readable domain names into IP addresses that computers understand.

> Think of DNS as the phone book of the internet. You know the name, DNS gives you the number (IP address).

---

## How DNS Works — Step by Step

When you type `comparify.bynassir.dev` in your browser:

```
User types: comparify.bynassir.dev
        ↓
1. Browser Cache        → did I visit this before?
        ↓ (if not found)
2. OS Cache             → does my laptop remember it?
        ↓ (if not found)
3. ISP / Router Cache   → does my internet provider know it?
        ↓ (if not found)
4. Root DNS Servers     → the actual source of truth
        ↓
IP address returned → request reaches the server
```

---

## DNS Record Types

| Record Type | Purpose | Example |
|---|---|---|
| **A Record** | Maps domain → IP directly | `bynassir.dev → 76.223.x.x` |
| **CNAME** | Maps domain → another domain (alias) | `comparify.bynassir.dev → cname.vercel-dns.com` |
| **MX Record** | For email routing | `mail.bynassir.dev → gmail servers` |
| **TXT Record** | Verification and security | Used by Google to verify domain ownership |

---

## Real Example — Comparify Deployment

When deploying `comparify.bynassir.dev` to Vercel:

1. Deployed app to Vercel → Vercel provided a CNAME target (e.g., `cname.vercel-dns.com`)
2. Went to Hostinger → created a subdomain `comparify`
3. Set record type as **CNAME** → pointed it to Vercel's target

**What this tells the internet:**
> "When someone looks up `comparify.bynassir.dev`, don't look for an IP — go look up `cname.vercel-dns.com` and use that IP instead."

```
User types: comparify.bynassir.dev
        ↓
Hostinger DNS: it's a CNAME → cname.vercel-dns.com
        ↓
Vercel DNS: it's IP → 76.223.x.x
        ↓
Request reaches Vercel → routed to your app
```

---

## DNS Caching and TTL

DNS lookup doesn't happen on every request — it is cached at multiple levels for performance.

**TTL (Time To Live)** — every DNS record has a TTL value.

> TTL = 3600 means *"cache this answer for 3600 seconds (1 hour), then ask again"*

### Why TTL matters in system design:

If you migrate your app from Vercel to AWS and update your DNS record:
- Users with the old IP cached won't get the new IP until their TTL expires
- This is why engineers **lower the TTL before migrations** — so changes propagate faster

---

## Key Takeaways

- DNS translates domain names → IP addresses
- CNAME is an alias record; A Record maps directly to an IP
- Caching happens at 4 levels: browser → OS → ISP → root DNS
- TTL controls how long a cached DNS entry is valid
- Lower TTL before migrations so DNS changes propagate faster

---

## System Design Relevance

| Concept | Where it appears in system design |
|---|---|
| DNS | Entry point of every system |
| CNAME | CDN and multi-region deployments |
| TTL | Migration strategy, blue-green deployments |
| DNS Caching | Latency optimization |
