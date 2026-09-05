# Components of an App — Detailed Notes

> **Parent Topic:** Scaling
> **Scope:** The distinct infrastructure components that make up a production web application, how they interact, and their role in scaling.

---

## 1. Overview

A production web application is never just "a server." It is a **system of interconnected components**, each with a distinct responsibility. Understanding each component is the first step to understanding where bottlenecks occur and how to scale.

### High-Level Architecture

```
                        Internet
                           │
                           ▼
                ┌─────────────────────┐
                │     CDN / Proxy     │  ← Serves static assets, caches responses
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │    Load Balancer    │  ← Distributes requests across app servers
                └──────┬──────┬───────┘
                       │      │
              ┌────────┘      └────────┐
              ▼                        ▼
   ┌──────────────────┐    ┌──────────────────┐
   │   App Server 1   │    │   App Server 2   │  ← Runs business logic
   │  (Frontend +     │    │  (Frontend +     │
   │   Backend code)  │    │   Backend code)  │
   └────────┬─────────┘    └────────┬─────────┘
            │                       │
            └──────────┬────────────┘
                       │
            ┌──────────▼──────────┐
            │    Cache Layer      │  ← Redis / Memcached (fast in-memory store)
            └──────────┬──────────┘
                       │
            ┌──────────▼──────────┐
            │     Database        │  ← Persistent data store (PostgreSQL, MongoDB)
            └─────────────────────┘
```

Each layer is independent — they can be scaled, upgraded, or replaced without affecting the others.

---

## 2. Server

The "server" in a web app is not a single thing. It is typically split into **multiple specialized server types**, each doing a distinct job.

---

### 2.1 Frontend Server

**What it is:**
The component responsible for **serving static assets** — HTML, CSS, JavaScript, images, fonts, and other files — directly to the client.

**What it does NOT do:**
- It does not execute business logic.
- It does not query databases.
- It does not generate personalised content (that's the backend/app server).

**Common software:**

| Software | Notes |
|---|---|
| **Nginx** | Most popular; extremely fast at serving static files; also used as reverse proxy |
| **Apache** | Classic, highly configurable; `.htaccess` support |
| **Caddy** | Modern, automatic HTTPS out of the box |
| **AWS S3 + CloudFront** | Fully managed static hosting + CDN |
| **Vercel / Netlify** | Managed frontend hosting platforms |

**Typical Nginx config for serving static files:**
```nginx
server {
    listen 80;
    server_name example.com;

    # Serve static files directly — no app server involved
    location /static/ {
        root /var/www/myapp;
        expires 1y;                        # Browser caches for 1 year
        add_header Cache-Control "public"; # Mark as publicly cacheable
        gzip_static on;                    # Serve pre-compressed .gz files
    }

    # Pass dynamic requests to the backend app
    location / {
        proxy_pass http://127.0.0.1:5000;  # Flask/Django running here
    }
}
```

**Why have a separate frontend server?**
- App servers (Flask, Django, Express) are **not optimized** for static file serving — they are slow at it.
- Nginx can serve thousands of static files per second with almost zero CPU.
- Offloading static files to Nginx (or a CDN) frees the app server to focus on dynamic requests.

**Static File Request Flow:**
```
User requests /static/logo.png
      │
      ▼
Nginx checks: is this a static file? ──YES──▶ Serve directly from disk
                                              App server never involved
      │
      NO
      ▼
Proxy to App Server (Flask/Django)
```

---

### 2.2 Database Server

**What it is:**
The component responsible for **storing, organizing, and retrieving persistent data**. Runs as a separate process (often on a separate machine) from the app server.

**Why a separate server?**
- Databases are resource-intensive (CPU for query processing, RAM for indexes and caches, Disk I/O for reads/writes).
- Running the database on the same machine as the app server means they compete for the same RAM and CPU.
- Separation allows **independent scaling** — you can add more RAM to the DB server without affecting app servers.

**What it stores:**
- User accounts, sessions, preferences
- Business data (orders, products, posts, transactions)
- Application state

**Database Server Architecture:**

```
App Server(s)                    Database Server
┌─────────────┐                  ┌─────────────────────────────┐
│ Flask App   │──SQL query──────▶│  PostgreSQL Process         │
│             │                  │  ├── Query Parser           │
│             │◀──result set─────│  ├── Query Planner          │
│             │                  │  ├── Executor               │
└─────────────┘                  │  └── Storage Engine         │
                                 │       ├── Data files (.db)  │
                                 │       ├── Indexes           │
                                 │       └── WAL (Write-Ahead  │
                                 │           Log)              │
                                 └─────────────────────────────┘
```

**Connection Pooling:**
- Creating a new DB connection for every request is expensive (involves TCP handshake + auth).
- A **connection pool** maintains a set of pre-established connections that are reused.
- Tools: **PgBouncer** (PostgreSQL), **SQLAlchemy** connection pool (Python), built-in pools in most ORMs.

```
App Server                 Connection Pool          Database
                           ┌──────────────┐
Request 1 ──────────────▶  │ Connection A │──────▶ PostgreSQL
Request 2 ──────────────▶  │ Connection B │──────▶ PostgreSQL
Request 3 ──── (waits) ──▶ │ Connection C │──────▶ PostgreSQL
                           │  [in use]    │
                           └──────────────┘
```

**Scaling the Database Server:**
- **Read replicas** — additional DB servers that are copies of the primary; serve SELECT queries.
- **Primary/Leader** — the single server that accepts all writes (INSERT/UPDATE/DELETE).
- **Sharding** — partitioning data across multiple DB servers by a key (e.g., user ID ranges).

```
        ┌─────────────┐
Writes ─▶   Primary   │─── replicates to ───▶ Replica 1 (reads)
        └─────────────┘                  ───▶ Replica 2 (reads)
                                         ───▶ Replica 3 (reads)
```

---

### 2.3 Load Balancer

**What it is:**
A component that **sits in front of multiple app servers** and distributes incoming requests across them, ensuring no single server is overwhelmed.

**Core responsibilities:**

| Responsibility | Description |
|---|---|
| **Traffic distribution** | Spread requests across backend servers using a chosen algorithm |
| **Health checking** | Regularly probe each server; remove unresponsive ones from rotation |
| **SSL/TLS termination** | Decrypt HTTPS at the LB; communicate with backends over HTTP (saves CPU) |
| **Session persistence** | Route the same user to the same server (sticky sessions) when needed |
| **Connection draining** | Allow in-flight requests to finish before taking a server offline |

**Load Balancing Algorithms:**

```
Round Robin:
  Request 1 ──▶ Server A
  Request 2 ──▶ Server B
  Request 3 ──▶ Server C
  Request 4 ──▶ Server A  (cycles back)

Least Connections:
  Server A: 10 active connections
  Server B: 3 active connections  ← next request goes here
  Server C: 7 active connections

IP Hash:
  User 203.x.x.x ──▶ always Server A  (deterministic)
  User 54.x.x.x  ──▶ always Server B
```

**Layer 4 vs Layer 7 Load Balancers:**

| Level | Operates On | Sees | Use Case |
|---|---|---|---|
| **L4 (Transport)** | TCP/UDP packets | Source IP, port | Simple, high-performance TCP routing |
| **L7 (Application)** | HTTP requests | URLs, headers, cookies, body | Route `/api/*` to API servers, `/static/*` to file servers |

**Health Check Example:**
```
Load Balancer checks every 30s:
  GET /health HTTP/1.1 Host: server-a

  Server A responds 200 OK  → ✅ Keep in rotation
  Server B responds 500     → ❌ Remove from rotation, alert ops
  Server B recovers → 200   → ✅ Add back to rotation
```

**When to introduce a Load Balancer:**
- When a **single app server** can no longer handle the request volume.
- When you need **zero-downtime deployments** (deploy to one server, keep others running, then rotate).
- When you need **geographic distribution** (different LBs in different regions).

---

### 2.4 Proxy

**What it is:**
A **proxy** is an intermediary server that sits between the client and the origin server, forwarding requests and responses. In web infrastructure, this is almost always a **reverse proxy** (the client talks to the proxy, not the origin directly).

**Forward Proxy vs Reverse Proxy:**

```
Forward Proxy:                     Reverse Proxy:
Client ──▶ Proxy ──▶ Internet      Client ──▶ Proxy ──▶ Your Servers
(client knows about proxy)         (client thinks proxy IS the server)
Use: Corporate firewalls, VPNs     Use: Web servers, CDNs, load balancers
```

**What a Reverse Proxy Provides:**

| Feature | Description |
|---|---|
| **Caching** | Store responses; serve repeat requests without hitting the origin |
| **SSL termination** | Handle HTTPS so backend servers don't have to |
| **Compression** | Compress responses (gzip/Brotli) before sending to client |
| **Rate limiting** | Block or throttle excessive requests per IP |
| **Security** | Hide backend server IPs; filter malicious requests |
| **Load balancing** | Nginx and HAProxy double as load balancers |
| **Request routing** | Route different paths to different backend services |

**Nginx as Reverse Proxy + Cache:**
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g;

server {
    location /api/ {
        proxy_pass http://backend:5000;
        proxy_cache my_cache;
        proxy_cache_valid 200 60s;     # Cache 200 responses for 60 seconds
        proxy_cache_key "$uri$args";   # Cache key = URL + query params
    }
}
```

**Request Flow with Caching Proxy:**
```
t=0s  User A: GET /api/products  ──▶  Proxy (MISS) ──▶ App Server ──▶ DB
                                       Proxy stores response for 60s

t=5s  User B: GET /api/products  ──▶  Proxy (HIT) ──▶ Cached response
                                       App Server and DB never involved

t=65s User C: GET /api/products  ──▶  Proxy (MISS, expired) ──▶ App Server
                                       Cache refreshed for next 60s
```

---

## 3. Network

The network is the physical and logical medium through which data travels between users and your servers. Developers have limited control over the user's network, but understanding it informs many optimization decisions.

---

### 3.1 Mobile vs Broadband

The performance characteristics of mobile networks vs fixed broadband are dramatically different and require different optimization strategies.

#### Broadband (Fixed Line / Fiber)

- **Technology:** DSL, Cable, Fiber-to-the-Home (FTTH)
- **Bandwidth:** 100 Mbps – 1 Gbps
- **Latency:** 5 – 20ms (very low and stable)
- **Reliability:** Stable, consistent
- **Data cost:** Usually unlimited, flat rate
- **User context:** Typically on a desktop or laptop, at home or office

#### Mobile (Cellular)

- **Technology:** 3G, 4G LTE, 5G
- **Bandwidth:** Varies widely (1 Mbps on 3G to 500 Mbps on 5G)
- **Latency:** 30 – 500ms (variable, depends on signal, tower load, handoff)
- **Reliability:** Variable — signal can drop, fluctuate, or hand off between towers
- **Data cost:** Often metered/capped — users pay per GB
- **User context:** Smartphone, outdoors or in transit, smaller screen

#### Comparison Table

| Factor | Broadband | 4G LTE | 3G | 5G |
|---|---|---|---|---|
| **Download speed** | 100–1000 Mbps | 10–50 Mbps | 1–5 Mbps | 100–500 Mbps |
| **Upload speed** | 50–500 Mbps | 5–20 Mbps | 0.5–2 Mbps | 50–200 Mbps |
| **Latency** | 5–20ms | 30–70ms | 100–500ms | 1–10ms |
| **Consistency** | High | Medium | Low | High |
| **Data cost** | Unlimited | Often capped | Capped | Often capped |

#### Developer Implications of Mobile Networks

**1. Minimize payload size**
- On a metered connection, a 5MB page costs real money.
- Users on 3G: a 2MB JavaScript bundle takes 3–16 seconds to download.
- Use image compression, lazy loading, and code splitting aggressively.

**2. Minimize round trips (requests)**
- 4G latency (~50ms) means each round trip adds 50ms.
- 20 requests = ~1 second of latency alone, before a byte is transferred.
- Bundle resources; use HTTP/2 multiplexing; reduce 3rd-party dependencies.

**3. Offline resilience**
- Mobile connections drop. A Service Worker can cache assets and serve the app offline.
- Progressive Web Apps (PWAs) use this to work even with intermittent connectivity.

**4. Responsive images**
- Don't serve a 1920px image to a 375px phone screen.
- Use `srcset` and `sizes` to serve appropriately-sized images per device.
```html
<img
  src="product-400.webp"
  srcset="product-400.webp 400w, product-800.webp 800w, product-1200.webp 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 1024px) 800px, 1200px"
  loading="lazy"
  alt="Product image"
>
```

**5. Consider the Network Information API**
- JavaScript can check the user's connection type and adapt:
```javascript
const connection = navigator.connection;
if (connection && connection.effectiveType === '2g') {
    // Load low-resolution images, disable autoplay, skip animations
}
```

---

## 4. Application

Beyond the infrastructure (servers, network), the **nature of your application** itself determines what kind of bottleneck you'll face and how to address it.

---

### 4.1 Data-Intensive Applications

**Definition:** Apps where the primary bottleneck is **reading and writing large amounts of data** from a database or data store.

**Examples:**
- Analytics dashboards (reading millions of rows to generate a chart)
- Social media feeds (assembling a personalised feed from many data sources)
- Search results pages (querying large indexes)
- Banking transaction history

**Characteristics:**
- Many and/or complex database queries per request
- Large result sets that need to be filtered, sorted, aggregated
- Database is the primary bottleneck (CPU, disk I/O, connection pool)

**Scaling Strategies:**

| Strategy | Description |
|---|---|
| **Database indexes** | Pre-sort data on queried columns to avoid full table scans |
| **Query optimization** | Rewrite slow queries; use EXPLAIN ANALYZE to find bottlenecks |
| **Read replicas** | Offload SELECT queries to replica servers |
| **Caching query results** | Store expensive query results in Redis; avoid re-computing every request |
| **Pagination** | Never return all rows — return pages of 20–100 at a time |
| **Denormalization** | Pre-compute and store aggregated data to avoid complex JOIN queries at runtime |
| **Async processing** | Generate complex reports in the background; return results when ready |

**Example — Caching a slow aggregation:**
```python
# Without caching: runs a heavy SQL aggregation on every request
def get_sales_dashboard():
    return db.query("SELECT region, SUM(amount) FROM orders GROUP BY region")

# With caching: runs heavy query once, serves from Redis for 5 minutes
def get_sales_dashboard():
    cache_key = "sales_dashboard"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    result = db.query("SELECT region, SUM(amount) FROM orders GROUP BY region")
    redis.setex(cache_key, 300, json.dumps(result))  # Cache for 300s
    return result
```

---

### 4.2 Image/Script-Intensive Applications

**Definition:** Apps where the primary bottleneck is **the size and number of static assets** — images, JavaScript bundles, CSS, video, fonts.

**Examples:**
- E-commerce product catalog (hundreds of product images per page)
- Photo sharing platforms (Instagram, Flickr)
- News/media sites (articles with many embedded images)
- Frontend-heavy SPAs with large JavaScript bundles

**Characteristics:**
- DB queries are fast; the bottleneck is asset delivery
- High bandwidth consumption
- Many HTTP requests for resources
- Frontend rendering performance affected by large JS bundles

**Scaling Strategies:**

| Asset | Strategies |
|---|---|
| **Images** | Use WebP/AVIF formats; compress; resize to display dimensions; use a CDN; lazy-load; use `srcset` |
| **JavaScript** | Code-split; tree-shake; minify; defer loading; use dynamic `import()`; move to SSR |
| **CSS** | Minify; remove unused CSS (PurgeCSS); inline critical CSS; load non-critical CSS async |
| **Fonts** | Subset fonts; use `font-display: swap`; preload key fonts; host locally instead of Google Fonts |
| **Video** | Use modern codecs (AV1, H.265); stream instead of download; use video CDN (Cloudinary, Mux) |
| **General** | Enable HTTP/2; use a CDN; set long-lived cache headers on all static assets |

**Image Optimization Pipeline:**

```
Original Image (5MB JPEG, 4000×3000px)
         │
         ▼
  ┌──────────────────────────────────────┐
  │  Image Processing Pipeline           │
  │  ├── Resize to required dimensions  │
  │  ├── Convert to WebP / AVIF         │
  │  ├── Compress (quality 75–85%)      │
  │  └── Generate srcset variants       │
  │      (400w, 800w, 1200w)            │
  └──────────────────────────────────────┘
         │
         ▼
  Optimized WebP (85KB, 1200×900px) ──▶ CDN ──▶ User
```

Tools: **Cloudinary**, **Imgix**, **Sharp** (Node.js), **Pillow** (Python), **Squoosh**, **Next.js Image Optimization**.

---

### 4.3 Matching App Type to Architecture

```
App Type         Primary Bottleneck     Key Architecture Choices
────────────────────────────────────────────────────────────────────
Data-intensive   Database               Read replicas, Redis cache,
                                        query optimization, async jobs

Image-intensive  Bandwidth / CDN        CDN, image optimization,
                                        lazy loading, WebP/AVIF

Script-intensive Browser CPU / JS       Code splitting, SSR/SSG,
                                        tree shaking, defer/async

Compute-heavy    CPU / GPU              Job queues (Celery, BullMQ),
(ML, encoding)                          dedicated worker servers
```

---

## 5. How the Components Interact — Full Request Flow

Putting it all together — the journey of a single request through the full stack:

```
User types URL in browser
        │
        ▼
[1] DNS Resolution  →  resolves example.com to CDN IP
        │
        ▼
[2] CDN Edge Node
    ├── Static asset? (CSS/JS/image)
    │     └── ✅ Serve from CDN cache → done in ~5ms
    └── Dynamic request?
          │
          ▼
[3] Load Balancer
    ├── Selects healthy App Server (e.g., Round Robin)
    └── Forwards request
          │
          ▼
[4] App Server (Flask / Django / Express)
    ├── Authenticates user (checks session/token)
    ├── Runs business logic
    ├── Checks Redis cache → HIT? Return cached data
    │                     → MISS? Query database
    │         │
    │         ▼
[5] Database Server
    ├── Executes SQL query
    ├── Returns result set
    └── App Server caches result in Redis
          │
          ▼
[6] App Server assembles response (HTML/JSON)
          │
          ▼
[7] Response passes back through Load Balancer
          │
          ▼
[8] Nginx (frontend server) may compress response
          │
          ▼
[9] Response arrives at browser → rendered for user
```

---

## 6. Key Takeaways

- A production app is **never just one server** — it is a system of specialized components, each with a distinct role.
- **Frontend server** (Nginx) handles static assets efficiently, freeing the app server for dynamic work.
- **Database server** is the most common scaling bottleneck — separate it from the app server and scale it independently with replicas.
- **Load balancer** enables horizontal scaling — add more app servers and distribute traffic across them.
- **Proxy** adds caching, security, compression, and routing between clients and servers.
- **Mobile networks** have much higher latency and lower bandwidth than broadband — optimize for mobile users by minimizing payload size and request count.
- **Data-intensive apps** bottleneck at the database → optimize with indexes, caching, and read replicas.
- **Image/script-intensive apps** bottleneck at asset delivery → optimize with CDN, compression, and modern formats.
- Understanding which component is the bottleneck guides every scaling decision.
