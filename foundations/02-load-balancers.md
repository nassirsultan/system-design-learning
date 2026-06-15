# 02 — Load Balancers

## The Problem — Why Do We Need Load Balancers?

When traffic grows, a single server starts to struggle. You have two options:

| Approach | Name | What it means | Problem |
|---|---|---|---|
| Add more CPU/RAM to same server | **Vertical Scaling (Scale Up)** | Make one server bigger | Expensive, has a hard limit, single point of failure |
| Add more servers | **Horizontal Scaling (Scale Out)** | Add multiple smaller servers | Who decides which server handles which request? |

> Horizontal scaling is preferred — but it creates a new problem: **traffic distribution**.
> This is exactly where a Load Balancer comes in.

---

## What is a Load Balancer?

A Load Balancer sits in front of your servers and distributes incoming requests across them.

```
                        → Server 1
User → Load Balancer    → Server 2
                        → Server 3
```

> The user only ever talks to the Load Balancer. They don't know how many servers are behind it.

---

## Load Balancing Strategies

| Strategy | How it works | Best used when |
|---|---|---|
| **Round Robin** | Takes turns equally across servers | All servers have equal capacity |
| **Least Connections** | Sends to server with fewest active requests | Requests have varying processing times |
| **Weighted Round Robin** | Servers with higher capacity get more requests | Servers have different specs |
| **IP Hashing** | Same user always goes to same server | Session consistency is needed |

### Round Robin — The Problem

```
Request 1 → Server 1 (done in 10ms)
Request 2 → Server 2 (heavy request, takes 5 seconds)
Request 3 → Server 3 (done in 10ms)
Request 4 → Server 1 (done quickly)
Request 5 → Server 2 ← still busy! Round Robin doesn't care
```

Round Robin is blind — it doesn't check how busy a server is.

### Least Connections — The Fix

```
Server 1 → 2 active requests
Server 2 → 8 active requests  (still busy)
Server 3 → 1 active request   ← new request goes here
```

Much smarter — always routes to the least busy server.

---

## Single Point of Failure

The Load Balancer itself can go down. If it does — everything goes down.

**Solution 1 — Active-Passive Setup (Classic)**

```
User → Virtual IP → Load Balancer 1 (active)   → Server 1
                    Load Balancer 2 (passive)   → Server 2
                                                → Server 3
```

- LB 1 is active — handling all traffic
- LB 2 is passive — watching LB 1
- If LB 1 goes down → **Virtual IP moves to LB 2 automatically**
- Users don't notice — they always connect to the same Virtual IP

This automatic switchover is called **Failover**.

**Solution 2 — Container Replicas (Modern)**

```
Kubernetes → runs 2 Load Balancer pods
           → if one pod dies, the other is already running
           → K8s automatically restarts the failed pod
```

Both approaches eliminate the single point of failure.

---

## Layer 4 vs Layer 7 Load Balancers

| Type | Operates at | Knows about | Speed | Intelligence |
|---|---|---|---|---|
| **Layer 4** | Network level | IP address, TCP port only | Very fast | Dumb — cannot read HTTP |
| **Layer 7** | Application level | HTTP headers, URLs, cookies | Slightly slower | Smart — understands requests |

### Layer 7 Example — Comparify App

```
User Request
      ↓
Layer 7 Load Balancer
      ↓
/api/**   → Spring Boot Backend (Render)
/**       → React Frontend (Vercel)
```

One entry point. Smart routing. Centralized security.

> **Spring Cloud Gateway is a Layer 7 Load Balancer** — it routes based on URL paths and handles security centrally. This is exactly what the API Gateway does in the `insurance-microservices-patterns` project.

---

## Key Takeaways

- Vertical scaling = bigger server, has limits and single point of failure
- Horizontal scaling = more servers, needs a load balancer
- Load balancer strategies: Round Robin, Least Connections, IP Hashing, Weighted
- Load balancer itself can fail — solved by Active-Passive setup or container replicas
- Layer 4 = fast but dumb (network level)
- Layer 7 = smart routing based on HTTP (application level)
- Spring Cloud Gateway = Layer 7 Load Balancer

---

## System Design Relevance

| Concept | Where it appears in system design |
|---|---|
| Horizontal Scaling | Every large scale system |
| Round Robin | Default strategy in most systems |
| Least Connections | Systems with heavy/unpredictable requests |
| Layer 7 LB | API Gateways, microservices routing |
| Active-Passive | High availability systems |
| Virtual IP + Failover | Zero downtime requirements |
