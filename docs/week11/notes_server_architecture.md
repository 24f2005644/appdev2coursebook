# Server Architecture Details — Detailed Notes

> **Parent Topic:** Scaling
> **Scope:** Deep dive into the server-side architectural choices that determine how well an application scales — load balancing, proxies, databases, and language/runtime decisions.

---

## 1. Overview

Server architecture is the set of **decisions about how your backend infrastructure is organized**. Poor architectural choices become scaling ceilings — hard limits beyond which the system cannot grow without fundamental redesign.

The four pillars of server architecture for scaling:

```
┌─────────────────────────────────────────────────────────┐
│                  SERVER ARCHITECTURE                    │
│                                                         │
│  ┌───────────────┐  ┌───────────┐  ┌────────────────┐  │
│  │ Load Balancer │  │   Proxy   │  │   Database     │  │
│  │               │  │   / CDN   │  │                │  │
│  │ Distributes   │  │ Caches &  │  │ Stores &       │  │
│  │ traffic       │  │ protects  │  │ retrieves data │  │
│  └───────────────┘  └───────────┘  └────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │               Language / Runtime                 │   │
│  │     Determines concurrency, throughput, speed    │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Server: Load Balancing

### 2.1 Basic Functionality

A **load balancer** is a server (hardware or software) that distributes incoming network traffic across a pool of backend servers. Its job is to ensure:

1. No single server is overwhelmed while others are idle.
2. If a server fails, traffic is rerouted to healthy servers.
3. New servers can be added (or removed) without downtime.

**Core Operation:**

```
                         ┌──────────────────────────────────┐
                         │         LOAD BALANCER            │
                         │                                  │
Incoming Requests        │  1. Receive request              │
─────────────────────▶   │  2. Select backend server        │
                         │     (via algorithm)              │
                         │  3. Forward request              │
                         │  4. Return response to client    │
                         └──────────┬───────────────────────┘
                                    │
               ┌────────────────────┼────────────────────┐
               ▼                    ▼                    ▼
         ┌──────────┐         ┌──────────┐         ┌──────────┐
         │ Server A │         │ Server B │         │ Server C │
         │  Flask   │         │  Flask   │         │  Flask   │
         │  :5000   │         │  :5001   │         │  :5002   │
         └──────────┘         └──────────┘         └──────────┘
```

**Load Balancing Algorithms — In Depth:**

#### Round Robin
Each server gets requests in strict rotation regardless of load.
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (back to start)
```
- ✅ Simple, fair distribution
- ❌ Ignores actual server load — one server might be handling a 10s request while getting more traffic

#### Weighted Round Robin
Servers get proportional share based on assigned weight.
```
Server A (weight=3): gets 3 out of every 6 requests
Server B (weight=2): gets 2 out of every 6 requests
Server C (weight=1): gets 1 out of every 6 requests
```
- ✅ Accounts for heterogeneous server capacities (e.g., one server has more RAM/CPU)
- ❌ Static weights — doesn't adapt to real-time load

#### Least Connections
New request goes to the server with the fewest active connections.
```
Server A: 12 active connections
Server B:  3 active connections  ← next request goes here
Server C:  8 active connections
```
- ✅ Adapts to actual load dynamically
- ✅ Best for requests with variable processing time
- ❌ Slightly more complex to implement

#### IP Hash (Sticky Sessions)
A hash of the client's IP address determines which server handles all their requests.
```
hash(203.0.113.42) % 3 = 1  →  always Server B
hash(198.51.100.7) % 3 = 0  →  always Server A
```
- ✅ Guarantees session affinity — the same user always hits the same server (useful if server-local sessions are used)
- ❌ Breaks load distribution if one IP sends much more traffic (e.g., corporate NAT — entire company appears as one IP)
- ❌ Doesn't rebalance if a server is added or removed

#### Least Response Time
Directs traffic to the server with the lowest average response time AND fewest active connections.
- ✅ Most performance-aware algorithm
- ❌ Requires ongoing response time tracking

**Health Checking:**

Load balancers continuously verify that backend servers are alive:

```
Load Balancer probes every 10s:
  GET /health HTTP/1.1  →  Server A
  
  Server A: HTTP 200 OK   →  ✅ In rotation
  Server B: TCP timeout   →  ❌ Removed from rotation, alert triggered
  Server B: HTTP 200 OK   →  ✅ Returned to rotation (after N consecutive successes)
```

**Passive health checks:** The LB monitors actual traffic responses — if too many 5xx errors, it marks the server unhealthy.
**Active health checks:** The LB proactively sends heartbeat requests to a `/health` endpoint.

---

### 2.2 Commercial Load Balancer Offerings

**Amazon Web Services (AWS)**

| Service | Layer | Best For |
|---|---|---|
| **Application LB (ALB)** | L7 (HTTP/HTTPS) | Web apps, microservices, WebSocket; path/header-based routing |
| **Network LB (NLB)** | L4 (TCP/UDP) | Extreme performance, static IPs, non-HTTP protocols |
| **Classic LB (CLB)** | L4 + L7 | Legacy — avoid for new deployments |
| **Gateway LB (GWLB)** | L3 (IP) | Network security appliances |

**ALB features:**
- Route `/api/*` to one target group, `/static/*` to another.
- Weighted routing (send 10% of traffic to a new version — blue/green deploy).
- Built-in WAF (Web Application Firewall) integration.
- Native integration with Auto Scaling Groups — automatically adds/removes instances.

```
ALB Architecture:
                          ┌─────────────────────┐
Internet ────────────────▶│  ALB (Application   │
                          │  Load Balancer)      │
                          └──────┬──────┬────────┘
                                 │      │
                    ┌────────────┘      └────────────┐
                    ▼                               ▼
         ┌──────────────────┐           ┌──────────────────┐
         │  Target Group 1  │           │  Target Group 2  │
         │  (API servers)   │           │  (Web servers)   │
         │  EC2 instances   │           │  EC2 instances   │
         └──────────────────┘           └──────────────────┘
```

**Google Cloud Platform (GCP)**

| Service | Layer | Notes |
|---|---|---|
| **Cloud Load Balancing (HTTP(S))** | L7 | Global anycast — single IP serves worldwide |
| **Network LB** | L4 | Regional, TCP/UDP |
| **Internal LB** | L4/L7 | Between services within VPC |
| **SSL Proxy / TCP Proxy** | L4 | SSL termination, TCP |

**GCP's key differentiator:** Global HTTP(S) Load Balancing uses anycast — clients connect to the nearest Google PoP, which routes to the nearest healthy backend. Truly global load balancing with a single IP.

**Microsoft Azure**
- **Azure Load Balancer** — L4, regional
- **Azure Application Gateway** — L7, WAF built-in, SSL termination
- **Azure Front Door** — Global, CDN + L7 LB + WAF

**Self-Hosted / Open Source**

| Software | Notes |
|---|---|
| **Nginx** | Excellent L7 LB + static file server + reverse proxy in one |
| **HAProxy** | Industry standard for high-performance TCP/HTTP load balancing |
| **Traefik** | Modern, container-native (Docker/Kubernetes), auto-discovers services |
| **Envoy** | High-performance proxy used in service meshes (Istio) |

---

## 3. Server: Proxy

### 3.1 Caching Proxies

A **caching proxy** (reverse proxy with caching) stores copies of backend responses and serves subsequent identical requests from cache — without contacting the origin server.

**Why it matters for scaling:**
- A single origin server might handle 500 req/s.
- With a caching proxy, 90% of requests might be cache hits → effectively handle 5,000+ req/s from the user's perspective.
- The origin only sees the remaining 10% (cache misses).

**Cache Key:**
The proxy uses a **cache key** to uniquely identify each cacheable response. Typically:
```
cache_key = method + scheme + host + path + query_string
           = "GET:https:example.com:/api/products:category=shoes"
```

**Cache Control via HTTP Headers:**

The origin server controls caching behaviour through response headers:

| Header | Purpose | Example |
|---|---|---|
| `Cache-Control: max-age=3600` | Cache this response for 3600 seconds | Static assets |
| `Cache-Control: no-cache` | Must revalidate with server before serving | User-specific pages |
| `Cache-Control: no-store` | Never cache (sensitive data) | Login pages, payment pages |
| `Cache-Control: public` | Can be cached by any proxy/CDN | Static files |
| `Cache-Control: private` | Only browser cache; not shared proxies | User's profile page |
| `Vary: Accept-Encoding` | Cache separate copies per encoding | Compressed vs uncompressed |
| `Surrogate-Control` | Specific to CDN/proxy cache (not sent to browser) | CDN-specific TTLs |

**Cache Invalidation:**
The hardest problem in caching — when should the cached copy be discarded?

```
Strategy 1: TTL (Time To Live)
  Cache expires after N seconds.
  Simple but potentially serves stale data.

Strategy 2: Purge on write
  When data changes, explicitly purge the cached URL.
  cache.delete("/api/products")
  Accurate but requires cache-aware application code.

Strategy 3: Cache-busting (for static assets)
  Include content hash in filename: app.a3f9b2c1.js
  When code changes, filename changes → browser fetches new file automatically.
  Old file URL remains cached safely (it's still the same content).
```

**Nginx Proxy Cache Configuration:**
```nginx
# Define cache storage
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=app_cache:10m    # 10MB zone for keys (metadata)
    max_size=2g                # Max 2GB of cached content on disk
    inactive=60m               # Remove items not accessed in 60 min
    use_temp_path=off;

server {
    location /api/ {
        proxy_pass          http://backend;
        proxy_cache         app_cache;
        proxy_cache_valid   200 302  10m;   # Cache 200/302 for 10 minutes
        proxy_cache_valid   404      1m;    # Cache 404 for 1 minute
        proxy_cache_use_stale error timeout updating; # Serve stale during error
        add_header          X-Cache-Status $upstream_cache_status; # HIT/MISS/BYPASS
    }
}
```

---

### 3.2 Content Delivery Networks (CDN)

A **CDN** is a globally distributed network of proxy/cache servers (called **edge nodes** or **Points of Presence / PoPs**) strategically placed around the world to serve content from locations geographically close to users.

**The Problem CDNs Solve:**

```
Without CDN:
User in Tokyo ─────────────────────── 150ms ──────────────────────▶ Origin (London)
              ◀─────────────────────── 150ms ─────────────────────── 
              Total: 300ms RTT before a byte arrives

With CDN:
User in Tokyo ── 5ms ──▶ CDN Edge (Tokyo) ── Cached response
              ◀─ 5ms ──
              Total: 10ms RTT — 30× faster
```

**CDN Architecture:**

```
                        Origin Server (London)
                               │
            ┌──────────────────┼───────────────────┐
            │                  │                   │
            ▼                  ▼                   ▼
   CDN PoP (Tokyo)    CDN PoP (New York)   CDN PoP (Mumbai)
   [Edge Cache]        [Edge Cache]         [Edge Cache]
         │                   │                    │
    Users in Asia      Users in USA         Users in India
    (~5ms latency)    (~10ms latency)       (~5ms latency)
```

**How a CDN request works:**

```
Step 1: User requests https://example.com/image.jpg
        DNS resolves to CDN anycast IP (routes to nearest PoP)

Step 2: CDN edge checks its local cache
        ├── CACHE HIT  → Return cached image immediately (5ms)
        └── CACHE MISS → Fetch from origin, cache it, return to user

Step 3: Subsequent requests for same resource served from edge
        (origin not involved again until TTL expires)
```

**What CDNs Cache:**

| Content Type | Cacheable? | Typical TTL |
|---|---|---|
| Images, fonts, icons | ✅ Yes | 1 year (with cache-busting) |
| CSS, JavaScript bundles | ✅ Yes | 1 year (with cache-busting) |
| HTML (static sites) | ✅ Yes | Minutes to hours |
| API responses (public data) | ✅ Partially | Seconds to minutes |
| User-specific API responses | ❌ No | — |
| Auth tokens, session data | ❌ Never | — |

**CDN Capabilities Beyond Caching:**

| Feature | Description |
|---|---|
| **DDoS protection** | Edge nodes absorb attack traffic before it reaches origin |
| **WAF (Web Application Firewall)** | Block SQLi, XSS, bot traffic at the edge |
| **TLS termination** | Handle HTTPS at the edge; free TLS certificates |
| **Image optimization** | Resize, compress, convert format on-the-fly (Cloudinary, Imgix) |
| **Edge computing** | Run JavaScript at the edge (Cloudflare Workers, Lambda@Edge) |
| **Geo-blocking** | Block or redirect requests from specific countries |
| **HTTP/2 & HTTP/3** | CDN speaks modern protocols to clients; may use HTTP/1.1 to origin |

**Major CDN Providers:**

| Provider | Strengths | Notes |
|---|---|---|
| **Cloudflare** | Security, DDoS protection, Workers (edge compute), free tier | Most popular |
| **AWS CloudFront** | Deep AWS integration, Lambda@Edge | Best for AWS-hosted apps |
| **GCP Cloud CDN** | Deep GCP integration, anycast | Best for GCP-hosted apps |
| **Fastly** | Developer-friendly, real-time purging, VCL customization | Popular with media companies |
| **Akamai** | Largest network, enterprise-grade | Used by major enterprises |
| **Cloudinary / Imgix** | Specialized image CDN with on-the-fly transforms | For image-heavy apps |

---

## 4. Server: Database (DB)

### 4.1 Choice of Database

**The single most impactful architectural decision** in backend engineering. There is no universally "best" database — the right choice depends on your data model, access patterns, and scaling requirements.

#### Relational Databases (SQL / RDBMS)

Store data in **tables with rows and columns**. Relationships between tables defined with foreign keys. Use SQL to query.

| Database | Best For | Key Characteristics |
|---|---|---|
| **SQLite** | Development, testing, embedded apps, small single-user tools | File-based (no server needed), no concurrent writes, zero setup |
| **PostgreSQL** | Most production web apps, complex queries, data integrity | Full SQL compliance, JSONB support, extensible, open source |
| **MySQL / MariaDB** | LAMP stack apps, high-read workloads | Slightly faster reads than PostgreSQL for simple queries; wide hosting support |
| **Microsoft SQL Server** | Enterprise / .NET ecosystems | Commercial, excellent tooling, Windows-native |

**When to use SQL:**
- Data has clear relationships (users → orders → products)
- ACID transactions are required (e.g., financial data — debit must match credit)
- Complex queries with JOINs, aggregations, window functions
- Data integrity and schema enforcement is important

#### Non-Relational Databases (NoSQL)

A broad category — different NoSQL databases solve very different problems.

| Type | Database | Best For | Notes |
|---|---|---|---|
| **Document** | **MongoDB** | Flexible schemas, JSON-like data, rapid iteration | Documents can have nested arrays/objects; horizontal sharding built-in |
| **Document** | **CouchDB** | Offline-first apps, sync to edge | Multi-master replication |
| **Key-Value** | **Redis** | Caching, sessions, pub/sub, leaderboards, queues | In-memory (must fit in RAM); extremely fast; optional persistence |
| **Key-Value** | **DynamoDB** | Serverless, massive scale, simple access patterns | AWS-managed; consistent performance at any scale; expensive for complex queries |
| **Wide Column** | **Apache Cassandra** | Massive write throughput, time-series, IoT | No single point of failure; eventual consistency; write-optimized |
| **Wide Column** | **HBase** | Hadoop ecosystem, analytical workloads | Built on HDFS |
| **Graph** | **Neo4j** | Social graphs, recommendation engines, fraud detection | Relationships are first-class; Cypher query language |
| **Search Engine** | **Elasticsearch** | Full-text search, log analytics | Not a primary database; use alongside PostgreSQL/MongoDB |
| **Time Series** | **InfluxDB / TimescaleDB** | Metrics, monitoring, IoT sensor data | Optimized for time-ordered data |

**Decision Framework:**

```
Does your data have complex relationships?
├── YES → SQL (PostgreSQL)
└── NO  → What's your primary access pattern?
          ├── Simple key lookups, caching → Redis
          ├── Flexible JSON documents, horizontal scale → MongoDB
          ├── Massive write throughput, time-series → Cassandra
          ├── Full-text search → Elasticsearch (alongside primary DB)
          └── Graph traversal → Neo4j
```

---

### 4.2 Scaling Issues: Reading vs Writing

This is the **most important concept** in database scaling. Reads and writes have fundamentally different scaling properties.

#### The Read/Write Ratio

Most web applications are **heavily read-biased**:

| Application Type | Typical Read:Write Ratio |
|---|---|
| Social media feed | 99:1 |
| E-commerce browsing | 95:5 |
| News site | 98:2 |
| Analytics dashboard | 99:1 |
| Real-time chat | 50:50 |
| Event logging / IoT | 5:95 |

#### Scaling Reads (Easy)

Reads can be distributed across multiple servers because **reading data doesn't change it**.

**Read Replicas:**
```
                    ┌──────────────────────┐
  Writes ──────────▶│   Primary (Leader)   │
                    │   Full read+write     │
                    └──────────┬───────────┘
                               │ replication
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
   │   Read Replica  │ │   Read Replica  │ │   Read Replica  │
   │   (Standby 1)   │ │   (Standby 2)   │ │   (Standby 3)   │
   │   SELECT only   │ │   SELECT only   │ │   SELECT only   │
   └─────────────────┘ └─────────────────┘ └─────────────────┘
          ▲                    ▲                    ▲
     Read traffic         Read traffic         Read traffic
```

- Replicas receive a stream of changes from the primary (**WAL — Write-Ahead Log** in PostgreSQL).
- Replication can be **synchronous** (replica must confirm write before primary acknowledges client — no data loss, slower) or **asynchronous** (primary acknowledges immediately, replica catches up — faster, tiny lag).
- Add replicas to handle growing read load — nearly infinitely scalable.

**Application-Level Read Routing:**
```python
# Route writes to primary, reads to replica
from sqlalchemy import create_engine

primary = create_engine("postgresql://primary-host/mydb")
replica = create_engine("postgresql://replica-host/mydb")

def get_user(user_id):
    return replica.execute("SELECT * FROM users WHERE id = %s", user_id)

def update_user(user_id, data):
    return primary.execute("UPDATE users SET ... WHERE id = %s", user_id)
```

#### Scaling Writes (Hard)

Writes are the fundamental challenge in database scaling because:

1. **Single source of truth:** All writes must go through a single primary to maintain consistency.
2. **Write conflicts:** If two servers both accept writes, they can make contradictory changes (e.g., two users simultaneously booking the last seat).
3. **ACID guarantees are difficult across distributed systems** (see: CAP theorem).

**Write Scaling Strategies:**

**1. Vertical Scaling (Short-term fix)**
- Give the primary more CPU, RAM, and faster SSDs.
- Quick but has a ceiling — there's only so big a machine you can get.

**2. Write Batching / Buffering**
- Instead of writing every event immediately, buffer them and write in batches.
- Risk: potential data loss if the server crashes before flush.
```python
# Instead of: 100 individual INSERTs
for event in events:
    db.execute("INSERT INTO analytics ...")

# Use bulk INSERT:
db.execute("INSERT INTO analytics VALUES %s", events)  # 1 query, 100 rows
```

**3. Sharding (Horizontal Partitioning)**
- Split the data across multiple databases by a **shard key**.
- Each shard accepts writes for its own subset of data.

```
Shard by user_id:
  user_id 0–999,999     → Database Shard 1  (primary + replicas)
  user_id 1M–1,999,999  → Database Shard 2  (primary + replicas)
  user_id 2M–2,999,999  → Database Shard 3  (primary + replicas)
```
- ✅ Write load distributed across multiple primaries
- ❌ Cross-shard queries are complex (JOIN across shards is very hard)
- ❌ Resharding when a shard grows too large is painful

**4. Event Sourcing / CQRS**
- **CQRS:** Command Query Responsibility Segregation — use completely separate data models for reads (query) and writes (command).
- **Event Sourcing:** Instead of updating rows, append events to an immutable log. State is derived by replaying events.
- Very scalable but architecturally complex.

**5. Use a Write-Optimized Database (for appropriate workloads)**
- Cassandra is designed for massive write throughput.
- InfluxDB / TimescaleDB for time-series write workloads.
- These use LSM trees (Log-Structured Merge-Tree) instead of B-trees — writes are always appends (fast), reads may be slower.

---

## 5. Server: Language

The choice of programming language and runtime determines **how fast requests are processed** and **how many can be handled concurrently**.

### 5.1 Interpreted vs Compiled

The execution model of a language has a direct impact on raw throughput.

#### Compiled Languages
Source code is **compiled ahead of time** (AOT) to native machine code. The CPU executes the binary directly — no interpreter overhead.

| Language | Compilation | Performance | Use In Web |
|---|---|---|---|
| **Go** | Compiled to native binary | Excellent | Growing (Caddy, Docker, Kubernetes all written in Go) |
| **Rust** | Compiled to native binary | Best-in-class | Systems programming, WebAssembly; Axum framework |
| **C / C++** | Compiled to native binary | Fastest possible | Nginx, Redis, databases written in C/C++ |
| **Java** | Compiled to JVM bytecode + JIT | Near-native | Large enterprise backends (Spring Boot) |
| **C#** | Compiled to CLR bytecode + JIT | Near-native | Microsoft ecosystem, .NET |
| **Kotlin** | Compiled to JVM bytecode + JIT | Near-native | Android, server-side (Ktor) |

#### JIT-Compiled Languages
Compiled to intermediate bytecode at build time, then **Just-In-Time compiled to native code at runtime** by the VM. Slower startup, near-native steady-state performance.

```
Java Source → javac → .class bytecode → JVM (JIT) → Native machine code
                                              ↑
                                     Optimizes hot code paths
                                     at runtime based on actual usage
```

#### Interpreted Languages
Source code is read and executed **line by line** at runtime by an interpreter. No compilation step — fast development but slower execution.

| Language | Interpreter | Performance | Notes |
|---|---|---|---|
| **Python** | CPython | ~50–100× slower than C | GIL limits true parallelism; most widely used in ML/web |
| **Ruby** | MRI Ruby | ~50× slower than C | Rails ecosystem |
| **PHP** | Zend Engine | ~20–30× slower | WordPress, Laravel; OPcache mitigates much of this |
| **JavaScript (V8)** | V8 (Chrome) | JIT-compiled — actually fast | Node.js; V8's JIT makes it much faster than typical interpreters |

**Does language performance matter in practice?**
- For I/O-bound web apps (waiting on DB, waiting on HTTP): **language speed barely matters** — the bottleneck is I/O, not CPU. Python and Ruby handle this fine.
- For CPU-bound workloads (image processing, ML inference, encryption): **language speed matters enormously** — use Go, Java, or offload to compiled C extensions.

```
Request timeline for typical web request:
  [App logic: 5ms] [DB query wait: 150ms] [Response: 5ms]
                          ↑
                   The bottleneck — not the language
```

---

### 5.2 Threading and Asynchronous Capabilities

How a language/runtime handles **concurrency** (many requests at the same time) is the most critical factor for web server scalability.

#### The Problem: Slow I/O

```
Typical request lifecycle:
  Receive request          (instant)
  Parse request            (1ms)
  Query database           ←──── 50–200ms of WAITING ────
  Process result           (5ms)
  Render response          (2ms)
  Send response            (instant)

During that 50–200ms wait: the CPU is idle — it could be handling other requests!
```

This is the I/O-bound problem. The solution is **concurrency** — handle another request while waiting for I/O.

#### Model 1: Multi-Process

Each request (or group of requests) is handled by a **separate OS process**. Each process has its own memory space.

```
Gunicorn (Python) with 4 workers:
  Master Process
    ├── Worker 1 (handles request A — waiting for DB)
    ├── Worker 2 (handles request B — processing)
    ├── Worker 3 (handles request C — waiting for DB)
    └── Worker 4 (handles request D — idle, ready)
```

- ✅ Simple, safe, no shared state issues
- ✅ One worker crashing doesn't affect others
- ❌ **High memory use** — each process duplicates memory (typical Flask worker: 50–100MB)
- ❌ Context switching between processes is expensive
- **Limit:** 4–32 workers per server (limited by RAM)

#### Model 2: Multi-Threaded

Multiple **threads** within one process share memory and handle requests concurrently. The OS scheduler switches between threads.

```
Java Servlet / Django threaded mode:
  Process (shared memory)
    ├── Thread 1 (request A — waiting for DB)
    ├── Thread 2 (request B — CPU processing)
    ├── Thread 3 (request C — waiting for DB)
    └── Thread 4 (request D — sending response)
```

- ✅ Lower memory than multi-process (shared heap)
- ✅ Fast communication between threads (shared memory)
- ❌ **Race conditions** — shared mutable state must be carefully synchronized
- ❌ **Python's GIL** — Global Interpreter Lock prevents true parallel execution of Python threads

**The Python GIL Problem:**
```
Python Multi-Threading — What you expect:
  Thread 1: ──────────────────────────────────
  Thread 2: ──────────────────────────────────
  (running in parallel)

Python Multi-Threading — Reality (CPython):
  Thread 1: ███░░░░███░░░███░░░░███░
  Thread 2: ░░░███░░░███░░░███░░░███
  (GIL allows only one thread to run Python bytecode at a time)

  Only useful for I/O-bound tasks (threads release GIL while waiting for I/O)
  Useless for CPU-bound tasks (GIL prevents true parallelism)
```

#### Model 3: Async / Event Loop (Most Scalable for I/O)

A **single thread** manages thousands of concurrent connections using an **event loop**. Instead of blocking while waiting for I/O, the thread registers a callback and moves on to the next request.

```
Node.js / Python asyncio event loop:

Event Loop (single thread):
  ├── Receive request A
  ├── Start DB query for A (non-blocking, register callback)
  ├── Receive request B                 ← immediately handles next request
  ├── Start DB query for B (non-blocking, register callback)
  ├── Receive request C
  ├── Start DB query for C (non-blocking, register callback)
  ├── ── DB result for B arrives ──▶ Process B, send response
  ├── ── DB result for A arrives ──▶ Process A, send response
  └── ── DB result for C arrives ──▶ Process C, send response
```

- ✅ **Thousands of concurrent connections on one thread**
- ✅ Very low memory overhead
- ✅ Excellent for I/O-bound workloads (most web apps)
- ❌ A single long-running CPU task **blocks the entire event loop** and freezes all other requests
- ❌ More complex programming model (async/await, callbacks)

**Python async example (FastAPI / aiohttp):**
```python
import asyncio
import aiohttp

async def fetch_user(user_id):
    # Does NOT block — event loop handles other requests while waiting
    async with aiohttp.ClientSession() as session:
        async with session.get(f"/api/users/{user_id}") as response:
            return await response.json()

async def handle_request(request):
    user = await fetch_user(request.params["id"])  # Non-blocking DB/HTTP call
    return {"user": user}
```

**Node.js example:**
```javascript
// Express with async/await — non-blocking I/O
app.get('/user/:id', async (req, res) => {
    const user = await db.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
    res.json(user);  // Event loop was free during DB wait
});
```

#### Model 4: Goroutines (Go) — The Best of Both Worlds

Go's **goroutines** are extremely lightweight "green threads" managed by the Go runtime, multiplexed onto OS threads automatically.

```
Go runtime with 8 OS threads:
  OS Thread 1: [goroutine A] [goroutine E] [goroutine I]
  OS Thread 2: [goroutine B] [goroutine F] [goroutine J]
  OS Thread 3: [goroutine C] [goroutine G] [goroutine K]
  ...
  
  100,000 goroutines running concurrently — each uses ~2KB RAM
  (Compare: 100,000 OS threads would require ~100GB RAM)
```

- ✅ True parallelism (no GIL)
- ✅ Handles both I/O-bound and CPU-bound workloads
- ✅ Trivially launch thousands of goroutines
- This is why Go is popular for high-performance web services.

**Concurrency Model Comparison:**

| Model | Language Examples | Concurrency | Memory | Complexity | Best For |
|---|---|---|---|---|---|
| Multi-process | Python (Gunicorn) | Limited (4–32) | Very High | Low | Simple apps, safety |
| Multi-threaded | Java, C# | Moderate | Medium | Medium | Mixed workloads |
| Async / Event loop | Node.js, Python asyncio | Very High (10K+) | Low | High | I/O-heavy apps |
| Goroutines | Go | Extremely High (100K+) | Very Low | Low | Any workload |

---

### 5.3 Programming Paradigms

The **programming paradigm** shapes how code is structured, and has real implications for testability, correctness, and scalability.

#### Imperative
- Tell the computer **exactly how to do it**, step by step.
- Explicit control flow: loops, conditionals, variable assignment.
- Most traditional code is imperative.

```python
# Imperative: explicit steps
result = []
for user in users:
    if user["active"]:
        result.append(user["name"].upper())
```

**Scaling relevance:** Imperative code with shared mutable state is prone to **race conditions** in concurrent systems.

#### Declarative
- Describe **what you want** — the system figures out how to achieve it.
- No explicit control flow — higher level of abstraction.

```sql
-- Declarative SQL: describe the result you want
SELECT UPPER(name) FROM users WHERE active = true;

-- HTML: declarative structure
<button type="submit">Save</button>

-- CSS: declarative styling  
.button { background: blue; color: white; }
```

**Scaling relevance:**
- Declarative queries (SQL) allow the **query optimizer** to choose the most efficient execution plan — the developer doesn't need to know how indexes work internally.
- Declarative infrastructure (Terraform, Kubernetes YAML) allows systems to self-heal and auto-scale.

#### Functional
- Functions are **first-class citizens** — they can be passed as arguments, returned, composed.
- **Pure functions** — given the same input, always return the same output. No side effects.
- **Immutability** — data is never modified in place; new data structures are created.

```python
# Functional: pure functions, no mutation
from functools import reduce

names = [user["name"].upper() for user in users if user["active"]]  # map + filter
total = reduce(lambda acc, x: acc + x, prices, 0)  # reduce
```

**Scaling relevance:**
- **Pure functions are inherently parallelizable** — no shared state means no race conditions.
- Immutability eliminates a whole class of concurrency bugs.
- Functional approaches make it easier to reason about behavior in distributed systems.
- Languages like Erlang (used in WhatsApp) and Elixir are built on functional principles specifically for scalable, fault-tolerant systems.

#### Object-Oriented (OOP)
- Organize code around **objects** that bundle data (attributes) and behaviour (methods).
- Key principles: Encapsulation, Inheritance, Polymorphism, Abstraction.

```python
class UserService:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache
    
    def get_user(self, user_id):
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            return cached
        user = self.db.query("SELECT * FROM users WHERE id = %s", user_id)
        self.cache.set(f"user:{user_id}", user, ttl=300)
        return user
```

**Scaling relevance:**
- OOP is the dominant paradigm in most web frameworks (Django, Spring, Rails, Laravel).
- Encapsulation helps manage complexity in large codebases.
- Mutable object state can lead to concurrency issues in multi-threaded environments.

**Paradigms in Real Systems:**
Most modern applications **mix paradigms** pragmatically:
- OOP for overall structure and dependency injection
- Functional techniques for data transformations (map, filter, reduce)
- Declarative SQL for database queries
- Imperative code for algorithms and performance-critical paths

---

## 6. Key Takeaways

- **Load balancing** enables horizontal scaling — choose the right algorithm (Round Robin for simplicity, Least Connections for variable-duration requests, IP Hash for sticky sessions).
- **AWS ALB and GCP Cloud Load Balancing** are the dominant managed offerings — ALB for flexible L7 routing, GCP for true global anycast.
- **Caching proxies** multiply effective server capacity — proper `Cache-Control` headers are what makes this work.
- **CDNs** solve the latency and origin-load problem globally — serve static assets from the nearest edge node.
- **Database choice** is the most consequential architectural decision — match the DB type to your data model and access pattern, not to trends.
- **Reads scale easily** (add replicas); **writes are hard to scale** (sharding, CQRS, batching).
- **Language performance matters less than concurrency model** for I/O-bound web apps — Python with asyncio or Go outperforms synchronous Java despite being "slower" languages.
- **The GIL** makes Python multi-threading ineffective for CPU-bound work — use multiple processes (Gunicorn) or async I/O (FastAPI) instead.
- **Async/event-loop** models (Node.js, asyncio, Go goroutines) are the most scalable for I/O-heavy workloads — thousands of concurrent connections on a single thread.
- **Functional programming principles** (pure functions, immutability) make concurrent code dramatically safer and easier to reason about.
