# Scaling — Detailed Notes



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Scaling — Detailed Notes**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

> **Parent Topic:** Performance
> **Scope:** Multi-user performance — how a web application behaves when many users access it simultaneously.

---

## 1. What is Scaling?

**Scaling** is the ability of a system to **handle increasing amounts of work** — more users, more requests, more data — without degrading in performance or availability.

Unlike **speed** (a single-user concern), scaling is about **many users at once**:

```
Speed:    1 user  ──▶ Server ──▶ Response in 200ms
                                ✅ Fast

Scaling:  1,000 users ──▶ Server ──▶ Response in 200ms each
                                     ✅ Scaled well

          10,000 users ──▶ Server ──▶ Response in 8,000ms or crash
                                      ❌ Not scaled
```

### Two Types of Scaling

| Type | Description | How |
|---|---|---|
| **Vertical Scaling** (Scale Up) | Add more power to the existing server — more CPU, more RAM | Upgrade the machine |
| **Horizontal Scaling** (Scale Out) | Add more servers and distribute load across them | Add more machines + load balancer |

```
Vertical Scaling:              Horizontal Scaling:
┌──────────────┐               ┌────┐ ┌────┐ ┌────┐
│  Server      │               │ S1 │ │ S2 │ │ S3 │
│  4 CPU → 16  │               └────┘ └────┘ └────┘
│  16GB → 64GB │                    ↑
└──────────────┘               Load Balancer distributes requests

Simple but has a ceiling         Complex but nearly unlimited capacity
```

---

## 2. Static vs Dynamic Content

The nature of your content fundamentally determines how hard it is to scale.

### 2.1 Static Content

**Definition:** Content that is the **same for every user** and does not change based on user input or database state. Generated at build time, not request time.

**Examples:**
- **Wikipedia** — Articles are mostly pre-rendered HTML. The same page is served to every visitor.
- **MDN (Mozilla Developer Network)** — Documentation pages are static HTML files.
- Personal blogs, marketing sites, documentation sites.

**Scaling characteristics:**
- Extremely easy to scale — files can be cached infinitely at every level (CDN, proxy, browser).
- No server-side computation per request.
- A single CDN can serve millions of static pages globally with minimal infrastructure.
- Cost is very low.

```
User in India ──▶ CDN Node (Mumbai) ──▶ cached HTML ──▶ Response < 10ms
User in USA   ──▶ CDN Node (Virginia) ──▶ cached HTML ──▶ Response < 10ms
(No origin server involved at all)
```

### 2.2 Dynamic Content

**Definition:** Content that is **generated per request**, tailored to the specific user, their session, or the current state of the database.

**Examples:**
- **E-commerce** (Amazon, Flipkart) — Product pages show personalised recommendations, live inventory, user-specific prices.
- **Learning platforms** (Coursera, Udemy) — Dashboard shows a user's enrolled courses, progress, certificates.
- Social feeds, banking dashboards, booking systems.

**Scaling characteristics:**
- Hard to scale — every request requires computation (database query, business logic, template rendering).
- Cannot be fully cached (user-specific data must be fresh).
- More servers, smarter architecture, and caching strategies all required.
- Cost is significantly higher.

```
User request ──▶ Load Balancer ──▶ App Server ──▶ Database Query
                                                        │
                                               Personalised response
                                               assembled & returned
```

### Static vs Dynamic: Comparison

| Aspect | Static | Dynamic |
|---|---|---|
| **Content** | Same for everyone | Personalised per user |
| **Generated** | At build time | At request time |
| **Caching** | Fully cacheable | Partially cacheable at best |
| **Scaling ease** | Trivial (CDN) | Complex (servers, DB, cache layers) |
| **Examples** | Wikipedia, MDN, blogs | E-commerce, dashboards, social feeds |
| **Cost** | Very low | High |

---

## 3. Response Under Load

### 3.1 Requests Per Second (RPS) / Throughput

**Throughput** is the number of requests a system can handle per unit of time.

- Also expressed as **RPS** (Requests Per Second) or **TPS** (Transactions Per Second).
- A server has a maximum throughput — beyond that, responses slow down or requests are dropped.

**What happens as load increases:**

```
Response Time
     │
     │                                          ╭──── Requests dropped / errors
     │                              ╭──────────╯
     │                   ╭─────────╯  ← Degradation zone
     │─────────────────╮╯
     │  Stable zone     │
     └──────────────────────────────────────────────────────▶ Requests/Second
     0       100       200       300       400       500
             ↑                   ↑
         Normal load          Capacity limit
```

- In the **stable zone**: response time is consistent regardless of load.
- In the **degradation zone**: response time increases as resources become saturated.
- Beyond capacity: requests queue up, timeouts occur, errors increase, server may crash.

### 3.2 Bottlenecks Under Load

When a system struggles under load, the bottleneck is usually one of:

| Bottleneck | Symptom | Solution |
|---|---|---|
| **CPU** | 100% CPU, slow processing | Scale horizontally, optimize code |
| **Memory** | OOM errors, swapping to disk | Add RAM, fix memory leaks, limit cache size |
| **Database** | Slow queries, DB connection pool exhausted | Query optimization, DB replicas, caching |
| **Network I/O** | High network latency, timeouts | CDN, HTTP/2, compression |
| **Disk I/O** | Slow file reads/writes | SSD, in-memory storage (Redis), caching |

---

## 4. Response Under Sudden Changes in Load

Gradual load increase is manageable. **Sudden spikes** are far more dangerous.

### Real-World Spike Scenarios

- **Flash sale** — Thousands of users simultaneously hit "Buy Now" at 12:00:00.
- **Viral content** — A post goes viral; site traffic multiplies 50× in minutes.
- **News event** — A breaking news site suddenly gets 100× its usual traffic.
- **DDoS attack** — Malicious flood of requests intended to overwhelm the server.

### Problems Caused by Spikes

```
Normal traffic:     ──────────────────────────────────
Spike:              ──────────────────╭──╮────────────
                                       │  │
                                 Spike arrives
                                       │
                    ┌──────────────────▼──────────────┐
                    │  Server overwhelmed              │
                    │  ├── Queue fills up             │
                    │  ├── Response times spike       │
                    │  ├── Errors / 503s              │
                    │  └── Possible crash             │
                    └─────────────────────────────────┘
```

### Strategies for Handling Spikes

| Strategy | Description |
|---|---|
| **Auto-scaling** | Cloud infrastructure automatically adds servers when load rises (AWS Auto Scaling, GCP Managed Instance Groups) |
| **Rate limiting** | Cap requests per user/IP to prevent any single source from overwhelming the server |
| **Queue-based architecture** | Put requests into a queue (e.g., RabbitMQ, Kafka); workers process at a controlled rate |
| **Circuit breaker** | Automatically reject or return cached responses when system is under stress, to prevent total collapse |
| **CDN offloading** | Serve as much as possible from CDN — takes a huge proportion of load off the origin server |
| **Graceful degradation** | Return a simplified version of the page under extreme load (e.g., disable recommendations, show cached data) |

---

## 5. Predictability of Load

Knowing *when* load will come is as important as being able to handle it.

### Types of Load Patterns

| Pattern | Description | Example |
|---|---|---|
| **Predictable cyclical** | Regular peaks at known times | News site peaks in morning; e-commerce peaks weekday evenings |
| **Predictable spikes** | Known events with predictable high load | Black Friday, IPL match streaming, exam results |
| **Unpredictable spikes** | Sudden, unforeseeable events | Viral content, breaking news, server going down causing retry storms |
| **Steady / flat** | Consistent load, little variation | Internal enterprise tools, B2B SaaS |

### Why Predictability Matters

- **Predictable:** You can pre-scale — provision extra servers *before* the event, not after.
- **Unpredictable:** You need **auto-scaling** and **resilience** built in from day one.

**Example:**
A Diwali sale announcement is sent at 10am for a 12pm sale start — the dev team can manually scale up the fleet at 11:30am. A news story going viral at 3am gives no such warning.

---

## 6. Components of an App

A production web application is made up of multiple components, each of which can be a scaling bottleneck.

```
                        ┌────────────────┐
  User ────────────────▶│  Load Balancer │
                        └───────┬────────┘
                                │  distributes to
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       ┌────────────┐   ┌────────────┐   ┌────────────┐
       │ App Server │   │ App Server │   │ App Server │
       │  (Flask /  │   │  (Flask /  │   │  (Flask /  │
       │  Node.js)  │   │  Node.js)  │   │  Node.js)  │
       └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                     ┌────────────────┐
                     │   Database     │
                     │  (PostgreSQL)  │
                     └────────────────┘
                              ▲
                     ┌────────────────┐
                     │  Cache Layer   │
                     │    (Redis)     │
                     └────────────────┘
```

### 6.1 Server

#### Frontend Server
- Serves **static assets** (HTML, CSS, JS, images) directly.
- Common choices: **Nginx**, **Apache**, **Caddy**.
- Should be configured to serve static files efficiently without involving the application layer.
- Handles SSL termination, HTTP/2, compression, and static file caching headers.

#### Database Server
- Stores and retrieves **persistent data**.
- Typically the **first bottleneck** in a scaling scenario.
- Runs separately from the app server to allow independent scaling.

#### Load Balancer
- Sits in front of multiple app servers and distributes incoming requests.
- Ensures no single server is overwhelmed.
- Also handles health checking — removes unhealthy servers from rotation automatically.

#### Proxy
- An intermediary between the client and the server.
- Can cache responses, compress content, handle SSL, and provide security filtering.
- Examples: Nginx as reverse proxy, HAProxy, Cloudflare.

---

### 6.2 Network

#### Mobile vs Broadband

| Factor | Mobile (4G/5G) | Broadband (Fiber) |
|---|---|---|
| **Bandwidth** | 10–100 Mbps | 100–1000 Mbps |
| **Latency** | 30–100ms | 5–20ms |
| **Variability** | High (signal strength) | Low (stable) |
| **Data cost** | Often metered | Usually unlimited |
| **Impact** | Crucial to minimize payload size | Less critical |

**Developer implications for mobile:**
- Compress all assets aggressively (Brotli/gzip).
- Use adaptive images (`srcset`) — don't serve 4K images to a 375px screen.
- Minimize number of requests (HTTP/2 helps, but mobile networks still suffer from latency).
- Consider offline support (Service Workers) for unreliable connections.

---

### 6.3 Application

#### Data Intensive vs Image/Script Intensive

Different apps have different bottlenecks:

| App Type | Primary Bottleneck | Scaling Strategy |
|---|---|---|
| **Data-intensive** (dashboards, analytics, feeds) | Database reads/writes | DB optimization, read replicas, caching query results |
| **Image-intensive** (photo sharing, e-commerce) | Bandwidth, storage | CDN for images, image optimization pipeline, object storage (S3) |
| **Script-intensive** (SPAs, complex UIs) | Browser CPU, JS parsing time | Code splitting, lazy loading, tree shaking, SSR |
| **Compute-intensive** (ML inference, video encoding) | CPU/GPU | Async job queues, dedicated compute workers |

---

## 7. Server Architecture Details

### 7.1 Load Balancing

**Basic Functionality:**
A load balancer distributes incoming HTTP requests across a pool of backend servers.

**Algorithms:**

| Algorithm | Description | Best For |
|---|---|---|
| **Round Robin** | Requests distributed sequentially to each server | Servers of equal capacity |
| **Least Connections** | New request goes to server with fewest active connections | Long-lived connections (WebSockets) |
| **IP Hash** | Same IP always goes to same server (sticky sessions) | Session-based apps without shared session store |
| **Weighted Round Robin** | Servers with more capacity get more requests | Mixed server capacities |
| **Random** | Random server selection | Simple, equal-capacity pools |

**Additional responsibilities of a Load Balancer:**
- **Health checks** — Periodically ping servers; remove failed ones from rotation automatically.
- **SSL termination** — Decrypt HTTPS at the load balancer; servers communicate over HTTP internally (saves CPU on app servers).
- **Connection draining** — When a server is removed, allow existing connections to finish before stopping traffic.

**Commercial Offerings:**

| Provider | Product | Notes |
|---|---|---|
| **AWS** | Elastic Load Balancer (ELB) | ALB (L7/HTTP), NLB (L4/TCP), CLB (legacy) |
| **GCP** | Cloud Load Balancing | Global anycast, HTTP(S) LB |
| **Azure** | Azure Load Balancer / Application Gateway | L4 and L7 options |
| **Cloudflare** | Load Balancing | Built into CDN layer |
| **Self-hosted** | Nginx, HAProxy | Full control, free, more ops overhead |

---

### 7.2 Proxy & CDN

#### Caching Proxy (Reverse Proxy)
- Sits between clients and the origin server.
- **Caches responses** — identical requests are served from cache without hitting the origin.
- Example: Nginx configured as a reverse proxy will cache the response to `GET /api/products` for 60 seconds — the 1,000th request in that minute never reaches the app server.

```
Request 1 ──▶ Proxy (cache MISS) ──▶ App Server ──▶ Response
                     │ stores response
Request 2 ──▶ Proxy (cache HIT)  ──▶ Cached response returned immediately
Request 3 ──▶ Proxy (cache HIT)  ──▶ Cached response returned immediately
   ...
Request 1000 ──▶ Proxy (cache HIT) ──▶ Cached response returned immediately
```

#### Content Delivery Networks (CDN)

A **CDN** is a geographically distributed network of servers (called **edge nodes** or **PoPs — Points of Presence**) that cache and serve content from locations close to the user.

**How a CDN works:**
```
Without CDN:
  User (Mumbai) ──── 150ms ────▶ Origin Server (US East)

With CDN:
  User (Mumbai) ─── 5ms ──▶ CDN Edge (Mumbai) ──▶ Cached response
                                    │ (if cache miss, fetches from origin once)
                                    └── 150ms ──▶ Origin Server (US East)
```

**What CDNs cache:**
- Static assets: images, CSS, JS, fonts, videos.
- Whole HTML pages (for static sites or cached dynamic pages).
- API responses (with appropriate `Cache-Control` headers).

**Benefits:**
- **Reduced latency** — Serve from the nearest edge node.
- **Reduced origin load** — Most traffic never reaches your server.
- **DDoS protection** — Edge nodes absorb attack traffic.
- **Automatic scaling** — CDN handles billions of requests globally.

**Major CDN providers:** Cloudflare, AWS CloudFront, GCP Cloud CDN, Fastly, Akamai.

---

### 7.3 Database (DB)

#### Choice of Database

| Database | Type | Best For | Notes |
|---|---|---|---|
| **SQLite** | SQL / Embedded | Development, small apps, single-server | Not suitable for concurrent writes; no network server |
| **PostgreSQL** | SQL / RDBMS | Most production apps, complex queries | Open source, feature-rich, scales well |
| **MySQL / MariaDB** | SQL / RDBMS | Web apps (LAMP stack), high read loads | Slightly simpler than PostgreSQL |
| **MongoDB** | NoSQL / Document | Flexible schemas, JSON-like data, rapid iteration | Horizontal scaling built-in (sharding) |
| **Redis** | NoSQL / In-memory | Caching, sessions, pub/sub, queues | Blazing fast, data must fit in RAM |
| **Cassandra** | NoSQL / Column-family | Massive write loads, time-series data | Eventually consistent, very scalable |
| **Elasticsearch** | Search Engine | Full-text search, log analytics | Part of ELK stack |

**Choosing a database for scaling:**
- If data is relational and consistency is critical → **PostgreSQL**
- If you need extreme write throughput → **Cassandra** or **MongoDB**
- If you need fast reads of frequently accessed data → **Redis** (as cache layer on top of primary DB)

#### Scaling Issues: Reading vs Writing

Databases have **asymmetric scaling challenges**:

**Reads** are easy to scale:
```
App Server ──▶ Primary DB (writes)
           ──▶ Read Replica 1 (reads)
           ──▶ Read Replica 2 (reads)
           ──▶ Read Replica 3 (reads)
```
- Add **read replicas** — copies of the database that serve SELECT queries.
- Most web apps are **read-heavy** (80–95% reads) → replicas dramatically reduce primary DB load.

**Writes** are hard to scale:
- All writes must go to the primary (to maintain consistency).
- Only one primary can accept writes (in traditional RDBMS setups).
- **Sharding** (partitioning data across multiple DB servers) is complex and expensive.
- This is why write-heavy systems often use NoSQL databases or event sourcing architectures.

**The N+1 Query Problem:**
A common anti-pattern that destroys DB performance:
```python
# Bad: N+1 queries
posts = db.query("SELECT * FROM posts")  # 1 query
for post in posts:
    author = db.query(f"SELECT * FROM users WHERE id={post.user_id}")  # N queries
# 101 posts = 102 queries!

# Good: JOIN or eager loading
posts = db.query("""
    SELECT posts.*, users.name 
    FROM posts JOIN users ON posts.user_id = users.id
""")  # 1 query
```

---

### 7.4 Server Language

The programming language powering your backend significantly affects raw performance and scalability.

#### Interpreted vs Compiled

| Type | Description | Examples | Performance |
|---|---|---|---|
| **Compiled** | Source code compiled to machine code ahead of time | Go, Rust, C, C++ | Fastest — no runtime overhead |
| **JIT Compiled** | Compiled to bytecode, JIT-compiled at runtime | Java, C#, Kotlin | Near-native speed |
| **Interpreted** | Code read and executed line by line at runtime | Python, Ruby, PHP | Slowest, but often fast enough |
| **Transpiled/VM** | Compiled to intermediate bytecode | JavaScript (V8), Python (CPython) | Middle ground |

**Python (Flask/Django) context:**
- Python is interpreted — slower raw throughput than Go or Java.
- **CPython has the GIL (Global Interpreter Lock)** — only one thread executes Python bytecode at a time.
- Mitigated by: running multiple worker processes (Gunicorn workers), async I/O (FastAPI + asyncio), or offloading heavy work to compiled extensions (NumPy).

#### Threading and Asynchronous Capabilities

How a server handles **concurrency** (many requests at the same time) is critical for scaling:

| Model | Description | Example | Scaling Behavior |
|---|---|---|---|
| **Multi-process** | Each request gets its own process | Gunicorn (pre-fork) | Safe, isolated, high memory use |
| **Multi-threaded** | Multiple threads share one process | Java Servlet, Django threads | Efficient memory, beware race conditions |
| **Async / Event loop** | Single thread handles many requests via non-blocking I/O | Node.js, Python asyncio, Go goroutines | Extremely efficient for I/O-bound workloads |

**The I/O-bound vs CPU-bound distinction:**
```
I/O-bound request (waiting for DB, HTTP calls):
  Thread/coroutine can be paused while waiting → async handles this perfectly

CPU-bound request (image processing, ML inference):
  CPU is busy the entire time → needs actual threads or processes, or offload to a worker queue
```

#### Programming Paradigms

| Paradigm | Description | Example |
|---|---|---|
| **Imperative** | Step-by-step instructions; tell the computer *how* to do it | Most procedural code |
| **Declarative** | Describe *what* you want; the engine figures out how | SQL, HTML, React JSX |
| **Functional** | Pure functions, immutability, no side effects | Haskell, parts of Python/JS |
| **Object-Oriented** | Organize code around objects with state and behaviour | Java, Python classes |

**Relevance to scaling:**
- **Pure functions** are trivially parallelizable (no shared state, no race conditions).
- **Immutability** eliminates a whole class of concurrency bugs.
- **Declarative SQL** lets the query optimizer choose the most efficient execution plan.

---

## 8. Monitoring and Measuring

You cannot scale what you cannot measure. Production monitoring is essential.

### 8.1 Server Logs

Every web server and application generates logs. Logs are the most basic form of observability.

**Types of logs:**

| Log Type | Contents | Example |
|---|---|---|
| **Access log** | Every HTTP request | `127.0.0.1 - "GET /api/products HTTP/1.1" 200 1234` |
| **Error log** | Errors, exceptions, warnings | `[ERROR] 500 Internal Server Error - database timeout` |
| **Application log** | Custom events from app code | `[INFO] User 4521 completed checkout for order #89234` |
| **Slow query log** | DB queries taking longer than threshold | `Query took 4.2s: SELECT * FROM products WHERE ...` |

**Nginx access log format:**
```
$remote_addr - [$time_local] "$request" $status $body_bytes_sent "$http_referer"
192.168.1.1   - [03/Sep/2026:10:30:00 +0530] "GET /api/users HTTP/2.0" 200 512 "-"
```

**Log analysis tools:**
- `grep`, `awk`, `cut` — basic command-line analysis
- GoAccess — real-time terminal web log analyzer
- ELK Stack — powerful analysis and visualization (see below)

---

### 8.2 Live Monitoring Tools

Logs are reactive (you analyze what already happened). Live monitoring tools give you **real-time visibility** into system health.

#### ELK Stack (Elasticsearch, Logstash, Kibana)

A powerful open-source logging and analytics pipeline:

```
Application / Servers
        │ logs
        ▼
┌──────────────┐     ┌──────────────────┐     ┌───────────────┐
│  Logstash    │────▶│  Elasticsearch   │────▶│    Kibana     │
│  (or Beats)  │     │  (storage +      │     │  (dashboard + │
│  Collect,    │     │   search index)  │     │   visualization│
│  Parse, Ship │     └──────────────────┘     └───────────────┘
└──────────────┘
```

- **Logstash / Filebeat:** Collects logs from servers, parses them, and ships to Elasticsearch.
- **Elasticsearch:** Indexes and stores logs; enables fast full-text search across billions of log lines.
- **Kibana:** Web UI for visualizing logs, building dashboards, setting alerts.

**Use cases:** Centralized logging, error rate tracking, latency percentile analysis, audit trails.

---

#### Prometheus + Grafana

The most popular open-source **metrics monitoring** stack:

```
App Servers (expose /metrics endpoint)
        │
        ▼
┌──────────────────┐
│   Prometheus     │  ← Scrapes metrics every 15s
│   (time-series   │     Stores: CPU%, request rate, error rate,
│    database)     │             DB connection pool usage, etc.
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    Grafana       │  ← Visualises metrics as dashboards
│   (dashboards +  │     Sends alerts (Slack, PagerDuty, email)
│    alerts)       │     when thresholds are breached
└──────────────────┘
```

**Prometheus metrics types:**
| Type | Description | Example |
|---|---|---|
| **Counter** | Monotonically increasing count | Total HTTP requests served |
| **Gauge** | Current value, can go up or down | Current memory usage |
| **Histogram** | Distribution of values (latency buckets) | Request duration in 0–100ms, 100–500ms, 500ms+ |
| **Summary** | Pre-calculated quantiles | p50, p95, p99 latency |

**Key metrics to monitor for scaling:**

| Metric | Why It Matters |
|---|---|
| **Request rate (RPS)** | Know your current load vs capacity |
| **Error rate (4xx/5xx)** | Rising errors signal a scaling problem |
| **p95 / p99 latency** | Tail latency reveals user experience at scale |
| **CPU utilization** | Identify compute bottlenecks |
| **Memory utilization** | Detect memory leaks |
| **DB connection pool** | Pool exhaustion = requests queuing / failing |
| **DB query duration** | Slow queries show up here before users complain |
| **Cache hit rate** | Low hit rate = more load on DB |

---

## 9. Key Takeaways

- **Scaling = multi-user performance.** Speed is for one user; scaling is for thousands.
- **Vertical scaling** (bigger machine) is simple but has a ceiling. **Horizontal scaling** (more machines) is complex but unlimited.
- **Static content scales trivially** (CDN); **dynamic content** is where the real scaling challenges live.
- Know your load patterns — **predictable spikes** can be pre-scaled; **unpredictable spikes** require auto-scaling and resilience.
- Every component — **load balancer, proxy/CDN, app server, database, language/runtime** — is a potential bottleneck.
- **Reads scale easily** (replicas); **writes are the hard part** in database scaling.
- **Language choice matters:** interpreted languages (Python) are slower and have concurrency limitations (GIL); async I/O (asyncio, Node.js) is key for I/O-bound scaling.
- **You cannot scale what you cannot see** — implement server logs and live monitoring (ELK + Prometheus/Grafana) from day one.
- **Measure first, then scale** — find your actual bottleneck before throwing more hardware at the problem.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

