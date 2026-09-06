# Summary — Short Notes (Quick Revision)



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Summary — Short Notes (Quick Revision)**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

> Complete revision cheat-sheet covering all topics: Performance, Scaling, and Caching.

---

## ⚡ PERFORMANCE

### What is Performance?
- Two dimensions: **Speed** (single user) + **Scaling** (many users)
- Measure first → optimize second. Never guess.

---

### Speed
- Time for one user to get a usable response
- **< 100ms** = instant | **> 3s** = user leaves
- **Perceived performance** (feels fast) ≠ **Actual performance** (is fast)
  - Improve perception with: skeleton screens, optimistic UI, progressive rendering

**5 Contributing Factors:**

| Factor | What affects it | Fix |
|---|---|---|
| Network | Latency (RTT), bandwidth | CDN, preconnect |
| # Requests | More requests = more round trips | Bundle, lazy-load |
| Response size | Large files = slow download | Minify, compress, WebP |
| HTTP version | HTTP/1.1 = sequential | Enable HTTP/2 (multiplexed) |
| Compression | Uncompressed = wasteful | Enable Brotli/gzip |

- **HTTP/2** multiplexes all requests over one connection → biggest protocol win
- **Brotli** compresses 10–25% better than gzip

---

### User Experience (UX)
- **UI** = how it looks | **UX** = how it feels
- Performance IS a UX problem — slow = bad UX regardless of visuals
- 53% of mobile users leave if load > 3s
- Bounce rate doubles from 1s → 3s load time
- Performance = accessibility (slow = exclusion for low-end devices / rural users)

---

### Tools & Measurement

**Lighthouse:**
- Chrome tool that audits a page under simulated Slow 4G + 4× CPU throttle
- Runs 50+ audits → scores 0–100 for Performance, Accessibility, Best Practices, SEO
- Run via: DevTools | CLI (`npx lighthouse <url>`) | CI/CD (`lighthouse-ci`)

**6 Performance Metrics:**

| Metric | Measures | Good | Weight |
|---|---|---|---|
| **FCP** | First content appears | < 1.8s | 10% |
| **Speed Index** | Pace of visual loading | < 3.4s | 10% |
| **LCP** ✅ CWV | Main content loaded | < 2.5s | 25% |
| **TTI** | Fully interactive | < 3.8s | 10% |
| **TBT** ✅ CWV proxy | Main thread blocked | < 200ms | 30% |
| **CLS** ✅ CWV | Layout stability | < 0.1 | 15% |

**Other Lighthouse Categories:**
- **Accessibility** — WCAG checks (alt text, contrast, ARIA labels, keyboard nav) — catches ~30% of issues
- **Best Practices** — HTTPS, no deprecated APIs, no console errors
- **SEO** — `<title>`, meta description, crawlability, font sizes

**Limitations of Automated Tools:**
- Lab ≠ Real world (simulated device/network)
- Score varies ±10–15 points between runs
- Doesn't test SPAs post-load, authenticated pages, or real user behaviour
- A high score can be gamed without improving real UX
- Fix: combine with Real User Monitoring (RUM)

---

## 📈 SCALING

### What is Scaling?
- Handling **increasing load** (more users, more requests) without performance degradation
- **Vertical** = bigger machine (ceiling exists) | **Horizontal** = more machines (unlimited)

**Static vs Dynamic Content:**

| | Static | Dynamic |
|---|---|---|
| Content | Same for all users | Personalised per user |
| Generated | Build time | Request time |
| Caching | Fully cacheable | Partially cacheable |
| Examples | Wikipedia, MDN | E-commerce, dashboards |
| Scale difficulty | Trivial (CDN) | Complex |

**Load Patterns:**
- **Predictable** → pre-scale (Diwali sale, exam results)
- **Unpredictable** → auto-scale + circuit breakers (viral content, breaking news)

**Response Under Load:**
- Stable zone → Degradation zone → Errors/Timeouts
- Bottlenecks: CPU, Memory, DB connection pool, Network I/O, Disk I/O

---

### Components of an App
```
CDN/Proxy → Load Balancer → App Servers → Cache (Redis) → Database
```

| Component | Role | Examples |
|---|---|---|
| **Frontend Server** | Serve static files efficiently | Nginx, Caddy, S3+CloudFront |
| **Load Balancer** | Distribute traffic across app servers | AWS ALB, Nginx, HAProxy |
| **Proxy** | Cache, compress, secure, route | Nginx, Varnish, Cloudflare |
| **Database** | Persist and retrieve data | PostgreSQL, Redis, MongoDB |

**Network: Mobile vs Broadband**
- Mobile: 30–500ms latency, often metered → minimize payload + requests
- Broadband: 5–20ms latency, unlimited → less critical to optimize
- Use `srcset`, Network Information API, lazy-load for mobile users

**App Types → Bottleneck:**
- Data-intensive → DB | Image-intensive → CDN/bandwidth | Script-intensive → JS bundle size

---

### Server Architecture Details

**Load Balancing Algorithms:**
- **Round Robin** — equal distribution, simple
- **Least Connections** — send to server with fewest active connections (best for variable requests)
- **IP Hash** — sticky sessions (same user → same server)
- **Weighted** — proportional to server capacity

**LB responsibilities:** Health checks, SSL termination, connection draining, session persistence

**AWS:** ALB (L7, path-based routing) | NLB (L4, extreme performance)
**GCP:** Cloud Load Balancing (global anycast — single IP, serves worldwide)
**Self-hosted:** Nginx, HAProxy, Traefik

---

**Proxy & CDN:**
- Reverse proxy sits between client and server — caches, compresses, secures
- **Cache key** = method + host + path + query string
- `Cache-Control` headers control what proxies cache and for how long
- **CDN** = geographically distributed cache nodes (PoPs) — serve from nearest location
  - Reduces latency, reduces origin load, absorbs DDoS
  - Major providers: Cloudflare, AWS CloudFront, GCP CDN, Fastly

---

**Database Scaling:**

| DB Type | Examples | Best For |
|---|---|---|
| SQL/RDBMS | PostgreSQL, MySQL | Relational data, ACID, complex queries |
| Document | MongoDB | Flexible schema, JSON |
| Key-Value | Redis | Caching, sessions, pub/sub |
| Wide-Column | Cassandra | Massive writes, time-series |
| Graph | Neo4j | Social graphs, recommendations |

**Reads vs Writes:**
- **Reads** → scale with read replicas (nearly unlimited)
- **Writes** → hard — single primary, scale via: batching, sharding, CQRS
- N+1 query problem: 1 query for list + N queries for each item → use JOIN or eager loading

---

**Server Language:**

| Type | Languages | Notes |
|---|---|---|
| Compiled | Go, Rust, C | Fastest — no interpreter overhead |
| JIT | Java, C# | Near-native via JVM/CLR |
| Interpreted | Python, Ruby | Slower but fast enough for I/O-bound |

- Python has the **GIL** — only one thread runs Python bytecode at a time
- For I/O-bound apps: language speed barely matters (bottleneck is DB wait)
- For CPU-bound apps: use compiled language or offload to worker queue

**Concurrency Models:**

| Model | Examples | Connections | Memory |
|---|---|---|---|
| Multi-process | Gunicorn | 4–32 workers | Very high |
| Multi-threaded | Java Servlet | Moderate | Medium |
| Async/Event loop | asyncio, Node.js | 10,000+ | Low |
| Goroutines | Go | 100,000+ | Very low |

**Programming Paradigms:**
- **Imperative** — explicit steps, mutable state → race condition risk
- **Declarative** — describe what (SQL, HTML) → optimizer does the how
- **Functional** — pure functions, immutable → parallelizable, no race conditions
- **OOP** — objects bundle data + behaviour → dominant in web frameworks

---

### Monitoring and Measuring

**3 Pillars of Observability:**
- **Logs** — what happened (events) → ELK Stack
- **Metrics** — how much / how fast (numbers) → Prometheus + Grafana
- **Traces** — how a request travelled → Jaeger, Zipkin

**Log Types:**
- **Access log** — every HTTP request (IP, method, path, status, size)
- **Error log** — exceptions, server errors
- **Application log** — custom app events (INFO/WARN/ERROR/CRITICAL)
- **Slow query log** — DB queries exceeding threshold

**Best practices:**
- Use **structured logging (JSON)** — machine-searchable by field
- **Centralize logs** — aggregate from all servers into one store
- **Rotate logs** — prevent disk exhaustion (logrotate)

**ELK Stack:**
```
Filebeat (ship) → Logstash (parse/enrich) → Elasticsearch (store/index) → Kibana (visualize)
```
- Elasticsearch: inverted index → search millions of logs in <1s
- Kibana: Discover (search), Visualize (charts), Dashboard, Alerts

**Prometheus + Grafana:**
```
App /metrics endpoint ← Prometheus scrapes every 15s → stores time-series → Grafana visualizes
```
- **4 metric types**: Counter (total), Gauge (current), Histogram (distribution), Summary (quantiles)
- **Alertmanager** → routes alerts to Slack / PagerDuty / email

**The 4 Golden Signals (Google SRE):**

| Signal | Example Metric |
|---|---|
| **Latency** | p95 / p99 response time |
| **Traffic** | Requests per second |
| **Errors** | 5xx rate |
| **Saturation** | CPU%, memory%, DB pool% |

---

## 🗄️ CACHING

### What is Caching?
- Store result of expensive operation → serve from cache on repeat requests
- Trade-off: **freshness vs performance**
- Invalidation is the hard part: *"There are only two hard things in CS: cache invalidation and naming things"*

**Cache Hierarchy (fastest → slowest):**
```
Browser → CDN/ISP Proxy → Reverse Proxy → App Cache (Redis) → DB Cache → Disk
```

---

### HTTP Caching Headers

**`Cache-Control` key directives:**

| Directive | Meaning |
|---|---|
| `max-age=N` | Cache for N seconds |
| `public` | Any cache (CDN + browser) can store |
| `private` | Browser only, not CDN |
| `no-store` | Never cache (payments, auth) |
| `no-cache` | Cache but must revalidate first |
| `immutable` | Never revalidate (hashed assets) |
| `stale-while-revalidate=N` | Serve stale, refresh in background |

**ETag — Conditional Requests:**
- Server sends `ETag: "abc123"` (content hash) with response
- Client sends `If-None-Match: "abc123"` on next request
- Server replies `304 Not Modified` if unchanged → **no body = bandwidth saved**

**Freshness Checking:**
- Fresh → serve from cache (no network)
- Stale + has ETag → send conditional request → 304 or new content
- Stale + no ETag → fetch full response

---

### Analytics Impact
- Cached requests never reach the server → access logs **undercount** real traffic
- Use **JavaScript-based analytics** (Google Analytics, Plausible) → fires in browser, unaffected by cache
- `Vary: Cookie` on high-traffic pages breaks CDN caching (every cookie = unique cache entry) ⚠️

---

### Flask Caching

**Setup:**
```python
from flask_caching import Cache
cache = Cache(app)  # config: CACHE_TYPE, CACHE_REDIS_URL, CACHE_DEFAULT_TIMEOUT
```

**`@cache.cached()` — for view functions:**
```python
@app.route('/api/products')
@cache.cached(timeout=60)          # caches full HTTP response
def get_products(): ...
```
- Default key = URL path → breaks with query params → use `key_prefix` function

**`@cache.memoize()` — for any function:**
```python
@cache.memoize(timeout=300)
def get_user(user_id): ...         # key = function name + arguments
cache.delete_memoized(get_user, user_id)  # invalidate on write
```

---

### Memoization
- Cache function return value keyed on arguments
- Function must be **pure/deterministic** (same input → same output always)
- `functools.lru_cache(maxsize=128)` — in-process, LRU eviction, args must be hashable
- `functools.cache` (Python 3.9+) — unlimited LRU
- Invalidate with `cache.delete_memoized(fn, *args)`

---

### Jinja Template Caching
- Jinja **auto-caches** compiled templates (bytecode) in memory → no re-parsing
- Cache entire view output: `@cache.cached()` on the route
- Cache partial template blocks with `{% cache 300, "key" %}...{% endcache %}`

---

### Caching Backends

| Backend | Speed | Shared? | Persistent? | Use When |
|---|---|---|---|---|
| **NullCache** | — | — | — | Testing / disable cache |
| **SimpleCache** | Fastest | ❌ Single process | ❌ | Dev, single worker |
| **FileSystemCache** | Medium | ✅ Same server | ✅ | Single server, survives restart |
| **RedisCache** | Very fast | ✅ All servers | ✅ configurable | **All production deployments** |

---

## 🎯 MEASURE, THEN OPTIMIZE

> The overriding principle across all topics.

| Step | Action |
|---|---|
| **1. Measure** | Lighthouse, Prometheus, ELK, access logs, RUM |
| **2. Find the bottleneck** | Is it DB? CPU? Network? JS? Images? |
| **3. Fix the specific bottleneck** | Don't optimize what isn't slow |
| **4. Measure again** | Verify the fix actually helped |
| **5. Monitor in production** | Set alerts so you know before users do |

---

## 👨‍💻 Developer Responsibilities

| Area | Developer Controls |
|---|---|
| **HTTP Headers** | `Cache-Control`, `ETag`, `Vary` — set correctly per resource type |
| **Asset optimization** | Minify, compress, use modern formats (WebP, Brotli) |
| **Application caching** | `@cache.cached()`, `@cache.memoize()`, Redis backend |
| **Database queries** | Indexes, N+1 avoidance, query optimization, read replicas |
| **Monitoring** | Set up Prometheus metrics, structured logging, alerts |
| **Cache invalidation** | Delete stale cache on writes, set appropriate TTLs |
| **Load testing** | Test before launch, not after — know your capacity limits |

---

## 📋 Quick Reference Card

```
SPEED WINS:
  ✅ Enable HTTP/2 on your server
  ✅ Enable Brotli/gzip compression
  ✅ Use a CDN for all static assets
  ✅ Set long Cache-Control headers on hashed static files
  ✅ Use WebP/AVIF for images with srcset
  ✅ Lazy-load images below the fold
  ✅ Code-split JavaScript — don't send it all at once
  ✅ Inline critical CSS

SCALING WINS:
  ✅ Put a load balancer in front of multiple app servers
  ✅ Separate DB server from app server
  ✅ Add read replicas for read-heavy apps
  ✅ Use Redis for session storage and app-level cache
  ✅ Use async I/O (asyncio/Node.js) for I/O-bound Flask apps
  ✅ Fix N+1 queries before they reach production

CACHING WINS:
  ✅ Cache DB query results with @cache.memoize()
  ✅ Cache view responses with @cache.cached()
  ✅ Use RedisCache backend in production
  ✅ Invalidate cache on writes (delete_memoized)
  ✅ Set correct Cache-Control headers on all responses
  ✅ Use ETag for frequently-revalidated resources

MONITORING WINS:
  ✅ Centralize logs (ELK or similar)
  ✅ Track the 4 Golden Signals (Latency/Traffic/Errors/Saturation)
  ✅ Set up alerts BEFORE problems occur
  ✅ Monitor p95/p99 latency — not just averages
  ✅ Check slow query log regularly
  ✅ Track cache hit rate — low hit rate = cache is ineffective
```

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
