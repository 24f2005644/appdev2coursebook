# Caching — Detailed Notes

> **Parent Topic:** Performance & Scaling
> **Scope:** What caching is, where it lives in the stack, how HTTP supports it, Flask-specific caching patterns, memoization, Jinja template caching, and the caching backend options.

---

## 1. What is Caching?

**Caching** is the process of **storing the result of an expensive operation so that future requests for the same result can be served faster** — by skipping the expensive work entirely.

### The Core Idea

```
Without Cache:
  Request ──▶ App Server ──▶ Database (100ms) ──▶ Process ──▶ Response
  Request ──▶ App Server ──▶ Database (100ms) ──▶ Process ──▶ Response
  Request ──▶ App Server ──▶ Database (100ms) ──▶ Process ──▶ Response
  (every request does the full work)

With Cache:
  Request 1 ──▶ App Server ──▶ Cache MISS ──▶ Database (100ms) ──▶ Store in cache
  Request 2 ──▶ App Server ──▶ Cache HIT  ──▶ Return cached result (1ms) ✅
  Request 3 ──▶ App Server ──▶ Cache HIT  ──▶ Return cached result (1ms) ✅
  (subsequent requests skip the expensive work)
```

### Why Cache?

| Problem | Cache Solution |
|---|---|
| Database query is slow | Cache the query result — skip the DB |
| Template rendering is CPU-intensive | Cache the rendered HTML — skip rendering |
| External API call is slow/rate-limited | Cache the API response — skip the call |
| Same data served to thousands of users | Cache once, serve many |
| Server cannot handle peak load | Cache reduces origin load dramatically |

### The Cache Trade-off

Caching introduces **staleness** — the cached copy may be out of date if the underlying data changes.

```
Freshness vs Consistency:
  ← More fresh                    More efficient →
  No cache ─────────────────────────────── Always cached
  Always stale        Sometimes stale    Always fresh

The developer's job: choose the right TTL (Time To Live) for each cache entry
based on how often data changes and how much staleness is acceptable.
```

---

## 2. Where to Cache

Caching can happen at multiple layers of the request-response pipeline. Each layer has different trade-offs.

### The Cache Hierarchy

```
Browser                              ← Client-side cache (fastest, most local)
    │
    ▼
CDN Edge Node / ISP Proxy            ← Shared proxy cache (serves many users)
    │
    ▼
Load Balancer / Reverse Proxy        ← Server-side proxy cache (Nginx, Varnish)
    │
    ▼
Application Server (Redis / Memory)  ← Application-level cache
    │
    ▼
Database Query Cache                 ← DB-level cache
    │
    ▼
Database (Disk)                      ← Slowest, source of truth
```

The **higher** the cache in the hierarchy, the faster the response and the lower the cost — but also the less control the developer has over it.

---

### 2.1 Client-Side Cache (Browser)

- Controlled via **HTTP response headers** (`Cache-Control`, `ETag`, `Expires`).
- Stored in the user's browser — local disk/memory.
- **Private** — only that user's browser can use it.
- Best for: static assets (CSS, JS, images), user-specific pages.

```
Browser cache hit:
  User requests /static/app.js
  Browser checks cache: found, not expired
  Returns from local disk — NO network request at all (0ms!)
```

### 2.2 CDN / ISP / Shared Proxy Cache

- A cache shared by many users in the same geographic area.
- Controlled by `Cache-Control: public` headers.
- **Public** — any user hitting the same CDN node benefits.
- Best for: static assets, public API responses, HTML for non-personalised pages.

```
CDN cache hit:
  1,000 users all request /api/products
  CDN serves all from cache after first request
  Origin server receives just 1 request
```

### 2.3 Reverse Proxy Cache (Server-Side)

- Nginx, Varnish, or similar — runs on the server infrastructure.
- The proxy sits between the internet and the app server.
- Caches entire HTTP responses.
- Best for: API responses, rendered HTML pages that don't change per-user.

### 2.4 Application Cache (Redis / In-Memory)

- Code running inside the application explicitly stores and retrieves data.
- Most flexible — developer has full control over what is cached, for how long, and invalidation.
- Backed by Redis, Memcached, or in-process memory.
- Best for: database query results, computed values, session data, API responses from external services.

### 2.5 Database Cache

- The database engine itself caches frequently accessed data (query results, buffer pool).
- Developer has limited control — it's automatic.
- PostgreSQL's shared_buffers, MySQL's InnoDB buffer pool.

### Where to Cache — Summary

| Layer | Location | Scope | Speed | Control |
|---|---|---|---|---|
| Browser | Client disk/memory | Private (1 user) | Fastest (0ms) | Via HTTP headers |
| CDN / Proxy | Edge node | Public (all users) | Very fast (5–20ms) | Via HTTP headers |
| Reverse Proxy | Server (Nginx) | Public / per-route | Fast (1–5ms) | Server config |
| App Cache (Redis) | Server memory | Custom | Fast (1–2ms) | Full code control |
| DB Cache | DB engine | Automatic | Moderate | Limited |
| DB Disk | Hard drive | — | Slow (50–200ms) | Source of truth |

---

## 3. Server Support for Caching

The **HTTP protocol** has built-in support for caching through response and request headers. The server controls caching behaviour by setting these headers.

### 3.1 HTTP `Cache-Control` Header

The most important caching header. Sent in the **HTTP response** from the server.

**Syntax:**
```http
Cache-Control: directive1, directive2, ...
```

**Key Directives:**

| Directive | Meaning | Use Case |
|---|---|---|
| `max-age=N` | Cache this response for N seconds | `max-age=3600` → cache 1 hour |
| `s-maxage=N` | Shared cache (CDN/proxy) TTL, overrides `max-age` for shared caches | CDN keeps longer than browser |
| `public` | Can be cached by any cache (browser + CDN + proxy) | Static assets, public API data |
| `private` | Only browser cache; NOT shared caches | User-specific data |
| `no-cache` | Cache the response BUT must revalidate before serving it | Data that changes frequently but ETag can still save bandwidth |
| `no-store` | Never cache this response anywhere | Sensitive data (payment, auth tokens) |
| `must-revalidate` | Once expired, must revalidate before serving stale | Strict freshness requirement |
| `stale-while-revalidate=N` | Serve stale while refreshing in background for N seconds | Great UX — no wait while updating |
| `immutable` | Response will never change; don't revalidate even if expired | Hashed static assets |

**Common Patterns:**

```http
# Long-lived static asset (1 year, immutable — file has hash in name)
Cache-Control: public, max-age=31536000, immutable
# e.g., app.a3f9b2c1.js — will never change; if content changes, filename changes

# API response — cached publicly for 60s, then revalidate
Cache-Control: public, max-age=60, stale-while-revalidate=30

# User's profile page — only browser cache, 5 minutes
Cache-Control: private, max-age=300

# Payment page — never cache
Cache-Control: no-store

# HTML page that changes often — must revalidate (ETag used for efficiency)
Cache-Control: no-cache
```

**Setting Cache-Control in Flask:**
```python
from flask import make_response

@app.route('/api/products')
def get_products():
    products = fetch_products()
    response = make_response(jsonify(products))
    response.headers['Cache-Control'] = 'public, max-age=60'
    return response

@app.route('/static/app.<hash>.js')
def static_js(hash):
    content = read_js_file(hash)
    response = make_response(content)
    response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'
    response.headers['Content-Type'] = 'application/javascript'
    return response
```

---

### 3.2 ETag (Entity Tag)

An **ETag** is a unique identifier (a hash or version number) representing the current version of a resource. It enables **conditional requests** — the browser can ask "has this changed?" without downloading it again if it hasn't.

**How ETag Works:**

```
Step 1: First Request
  Client: GET /api/products HTTP/1.1

  Server: HTTP/1.1 200 OK
          ETag: "abc123def456"         ← hash of the response content
          Cache-Control: no-cache
          Content: [products JSON]
  
  Browser stores: ETag value + response content

Step 2: Subsequent Request (after cache expires or with no-cache)
  Client: GET /api/products HTTP/1.1
          If-None-Match: "abc123def456"  ← sends back the stored ETag

  Server (data NOT changed):
          HTTP/1.1 304 Not Modified      ← no body — saves bandwidth!
          ETag: "abc123def456"
  
  Server (data HAS changed):
          HTTP/1.1 200 OK
          ETag: "xyz789new456"           ← new ETag
          Content: [updated products JSON]
```

**Benefits:**
- **Bandwidth savings**: 304 response has no body — just headers.
- **Accuracy**: Unlike `max-age`, ETag works even when you can't predict how often data changes.
- **Works with `no-cache`**: Browser always revalidates, but avoids re-downloading unchanged content.

**Generating ETags in Flask:**
```python
import hashlib
import json
from flask import request, make_response, jsonify

@app.route('/api/products')
def get_products():
    products = fetch_products()
    data = json.dumps(products, sort_keys=True)
    
    # Generate ETag from content hash
    etag = hashlib.md5(data.encode()).hexdigest()
    
    # Check if client sent a matching ETag
    if request.headers.get('If-None-Match') == etag:
        return '', 304  # Not Modified — no body needed
    
    response = make_response(jsonify(products))
    response.headers['ETag'] = etag
    response.headers['Cache-Control'] = 'no-cache'  # Must revalidate, but ETag saves bandwidth
    return response
```

**Weak vs Strong ETags:**
```http
ETag: "abc123"     ← Strong: byte-for-byte identical
ETag: W/"abc123"   ← Weak: semantically equivalent (content may differ slightly, e.g., whitespace)
```

---

### 3.3 Freshness Checking

**Freshness** is the determination of whether a cached response is still valid (fresh) or needs to be re-fetched from the server (stale).

**The Freshness Calculation:**
```
response_is_fresh = (current_time < date_of_response + max_age)

Example:
  Response received at:  10:00:00
  max-age:               3600 seconds (1 hour)
  Expires at:            11:00:00
  
  Current time 10:30:00  → FRESH (serve from cache)
  Current time 11:15:00  → STALE (must revalidate or re-fetch)
```

**The Full Freshness Decision Tree:**
```
Browser receives request for cached resource
            │
            ▼
   Is there a cached copy?
   ├── NO  ──▶ Fetch from server
   └── YES
          │
          ▼
   Cache-Control: no-store?
   ├── YES ──▶ Never cached — fetch from server
   └── NO
          │
          ▼
   Cache-Control: no-cache?
   ├── YES ──▶ Must revalidate — send conditional request (If-None-Match)
   └── NO
          │
          ▼
   Is the cached copy still fresh? (within max-age)
   ├── YES ──▶ Serve from cache (no network request!)
   └── STALE
          │
          ▼
   Does it have an ETag or Last-Modified?
   ├── YES ──▶ Send conditional request:
   │          If-None-Match: "etag" or If-Modified-Since: date
   │          ├── 304 Not Modified ──▶ Serve from cache (no body downloaded)
   │          └── 200 OK ──────────▶ Update cache with new response
   └── NO  ──▶ Fetch fresh copy from server
```

**`Last-Modified` header (alternative to ETag):**
```http
HTTP/1.1 200 OK
Last-Modified: Wed, 03 Sep 2026 10:00:00 GMT

# Client revalidates with:
GET /api/products HTTP/1.1
If-Modified-Since: Wed, 03 Sep 2026 10:00:00 GMT

# If unchanged:
HTTP/1.1 304 Not Modified
```

**ETag vs Last-Modified:**

| | ETag | Last-Modified |
|---|---|---|
| **Based on** | Content hash / version | Timestamp |
| **Granularity** | Exact content match | 1-second resolution |
| **Server cost** | Must compute hash | Just check mtime |
| **Reliability** | High — detects any change | Lower — same-second changes missed |
| **Preferred** | ✅ Yes (more accurate) | Fallback |

---

## 4. Impact on Website Popularity / Analytics

Caching has a **significant but often overlooked side effect**: it distorts analytics and popularity metrics.

### 4.1 The Problem

When a CDN or browser cache serves a request, **the origin server never sees that request**. Therefore:
- Web server access logs are incomplete — they only show cache misses.
- Analytics tools that rely on server logs (AWStats, GoAccess) undercount actual traffic.
- Page view counts, visitor counts, and popular content rankings are skewed.

```
Reality: 10,000 users viewed /api/products today

Server access log shows:
  /api/products — 800 requests
  (9,200 were served from CDN/browser cache — never hit the server)
```

### 4.2 The Solutions

**Client-side analytics (JavaScript-based):**
- Tools like Google Analytics, Plausible, Mixpanel inject a JavaScript snippet that fires in the user's browser.
- Runs **after** the cache has served the page — counts every real user, regardless of caching.
- ✅ Accurate for user counts and page views
- ❌ Blocked by ad blockers; doesn't work in browsers with JS disabled

```html
<!-- Fires in the browser on every page view, even if page came from cache -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_TRACKING_ID"></script>
```

**Cache-Control for analytics endpoints:**
- The analytics beacon itself (`/analytics`, `/track`) should **never be cached**:
```http
Cache-Control: no-store
```

**CDN real-user metrics:**
- Most CDNs (Cloudflare, Fastly) provide their own analytics that count all edge requests — including cached ones.
- ✅ Shows full traffic picture including cache hits

**`Vary` header considerations:**
- The `Vary` header tells shared caches that responses vary based on certain request headers.
```http
Vary: Accept-Encoding   ← Cache separate copies for gzip vs non-gzip clients
Vary: Accept-Language   ← Cache separate copies per language
Vary: Cookie            ← Different response per cookie value (DANGEROUS — explodes cache size)
```
- Using `Vary: Cookie` or `Vary: Authorization` on high-traffic endpoints can effectively **bypass CDN caching** (every unique cookie = separate cache entry).

### 4.3 Cache Impact on A/B Testing

- If a page is cached, all users get the same cached variant — the A/B test is broken.
- Solution: Run A/B logic at the edge (CDN edge worker) or ensure A/B pages are not cached / use `Vary: Cookie` carefully.

---

## 5. Flask Caching

Flask provides a caching extension — **Flask-Caching** — that integrates cleanly with the application and supports multiple backends.

### 5.1 Module Integration

**Install:**
```bash
pip install Flask-Caching
```

**Basic Setup:**
```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)

# Configuration
app.config.from_mapping({
    'CACHE_TYPE': 'RedisCache',           # Backend type
    'CACHE_REDIS_URL': 'redis://localhost:6379/0',
    'CACHE_DEFAULT_TIMEOUT': 300,         # Default TTL: 5 minutes
})

cache = Cache(app)
```

**Alternatively, with `init_app` pattern (for large apps):**
```python
cache = Cache()

def create_app():
    app = Flask(__name__)
    app.config['CACHE_TYPE'] = 'RedisCache'
    cache.init_app(app)
    return app
```

---

### 5.2 Caching View Functions (`@cache.cached`)

The simplest use case: cache the **entire response** of a route for a given TTL.

```python
@app.route('/api/products')
@cache.cached(timeout=60)  # Cache for 60 seconds
def get_products():
    # This function body is only executed on cache MISS
    products = db.session.query(Product).all()
    return jsonify([p.to_dict() for p in products])
```

**How it works internally:**
1. On first request: Flask executes the view, stores the response in cache with a key derived from the URL.
2. On subsequent requests (within 60s): Flask returns the cached response immediately — the view function never runs.

**Custom timeout per route:**
```python
@app.route('/api/homepage-stats')
@cache.cached(timeout=3600)  # 1 hour — data doesn't change often
def homepage_stats():
    ...

@app.route('/api/live-scores')
@cache.cached(timeout=10)  # 10 seconds — changes frequently
def live_scores():
    ...
```

---

### 5.3 Cache Key Mapping

The **cache key** is the identifier used to store and retrieve a cached value. Flask-Caching generates a default key from the request URL, but this can be customized.

**Default cache key (for view functions):**
```
view/{view_function_name}/{request.path}
e.g.: view/get_products//api/products
```

**Problem with default key:**
```python
@app.route('/api/products')
@cache.cached(timeout=60)
def get_products():
    category = request.args.get('category')  # ?category=shoes
    return products_for_category(category)

# Cache key: view/get_products//api/products
# Same key regardless of ?category=shoes or ?category=bags!
# First request caches shoes, next user asking for bags gets shoes ❌
```

**Solution: Custom cache key function:**
```python
def make_cache_key(*args, **kwargs):
    # Include query string in key
    return f"products:{request.query_string.decode()}"
    # → "products:category=shoes"
    # → "products:category=bags" (different key!)

@app.route('/api/products')
@cache.cached(timeout=60, key_prefix=make_cache_key)
def get_products():
    category = request.args.get('category')
    return products_for_category(category)
```

**Key prefix as string (simple customization):**
```python
@app.route('/api/featured')
@cache.cached(timeout=300, key_prefix='featured_products')
def featured_products():
    ...
```

**Per-user caching (include user ID in key):**
```python
def user_cache_key():
    user_id = get_current_user_id()
    return f"user_dashboard:{user_id}"

@app.route('/dashboard')
@cache.cached(timeout=120, key_prefix=user_cache_key)
def dashboard():
    ...
```

**Manual cache operations:**
```python
# Manually get/set/delete cache entries
cache.set('my_key', my_value, timeout=300)
value = cache.get('my_key')   # Returns None if expired or not set
cache.delete('my_key')        # Invalidate a specific entry
cache.clear()                 # Clear ALL cache entries (use carefully!)
```

---

### 5.4 Caching Non-View Functions (`@cache.memoize`)

`@cache.cached` is for view functions (it caches based on request URL).
`@cache.memoize` is for **any function** — it caches based on the function's arguments.

```python
@cache.memoize(timeout=600)
def get_user_profile(user_id):
    # Slow DB query — only runs on cache miss
    user = db.session.query(User).get(user_id)
    return user.to_dict()

# Usage:
profile = get_user_profile(4521)  # Cache MISS — queries DB
profile = get_user_profile(4521)  # Cache HIT  — returned from cache instantly
profile = get_user_profile(9876)  # Cache MISS — different user_id = different cache key
```

**Invalidating memoized functions:**
```python
# Delete cache for a specific set of arguments
cache.delete_memoized(get_user_profile, 4521)

# Delete cache for all argument combinations of the function
cache.delete_memoized(get_user_profile)
```

**Practical pattern — invalidate on update:**
```python
@cache.memoize(timeout=600)
def get_user_profile(user_id):
    return db.session.query(User).get(user_id).to_dict()

def update_user_profile(user_id, new_data):
    db.session.query(User).filter_by(id=user_id).update(new_data)
    db.session.commit()
    cache.delete_memoized(get_user_profile, user_id)  # Invalidate stale cache
```

---

## 6. Memoization

**Memoization** is a specific caching technique where the return value of a function is cached based on its input arguments. It's a form of **function-level caching**.

### 6.1 What is Memoization?

```
Traditional function call:
  fibonacci(35) → [computes recursively] → 9227465  (takes seconds)
  fibonacci(35) → [computes recursively] → 9227465  (takes seconds again)

Memoized function call:
  fibonacci(35) → [computes, stores result] → 9227465  (slow, first time)
  fibonacci(35) → [cache HIT] → 9227465               (instant, subsequent times)
```

**Core properties:**
- The function must be **pure** (or at least deterministic): same inputs must always produce the same output.
- Each unique combination of arguments gets its own cache entry.
- Cached in memory (or Redis for Flask-Caching) rather than a database.

### 6.2 Memoizing with Function Arguments

**Simple Python memoization with `functools.lru_cache`:**
```python
from functools import lru_cache

@lru_cache(maxsize=128)  # Cache up to 128 unique argument combinations
def compute_tax(amount, tax_rate):
    # Expensive calculation
    return amount * tax_rate

compute_tax(1000, 0.18)  # Computed and cached → 180.0
compute_tax(1000, 0.18)  # Cache hit → 180.0 (instant)
compute_tax(2000, 0.18)  # Different args → computed → 360.0
compute_tax(1000, 0.28)  # Different args → computed → 280.0
```

**`lru_cache` — Least Recently Used:**
- When the cache is full (`maxsize=128`), the **least recently used** entry is evicted to make room.
- `maxsize=None` → unlimited cache (use cautiously — can grow to fill RAM).
- All arguments must be **hashable** (no lists or dicts as arguments — use tuples instead).

```python
# ❌ This won't work — list is not hashable
@lru_cache(maxsize=128)
def process(items: list):  # Error!
    ...

# ✅ Convert to tuple
@lru_cache(maxsize=128)
def process(items: tuple):
    ...

# Call with: process(tuple([1, 2, 3]))
```

**`functools.cache` (Python 3.9+) — Unlimited LRU:**
```python
from functools import cache

@cache  # Equivalent to lru_cache(maxsize=None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(100)  # Without cache: exponential time. With cache: linear time.
```

**Flask-Caching `@cache.memoize` with arguments:**
```python
@cache.memoize(timeout=300)
def get_products_by_category(category, page, per_page):
    # Cache key automatically includes all arguments:
    # "get_products_by_category:shoes:1:20"
    return db.session.query(Product)\
        .filter_by(category=category)\
        .paginate(page=page, per_page=per_page)\
        .items

# Each unique (category, page, per_page) combination cached separately
get_products_by_category('shoes', 1, 20)   # Miss → DB → Cached
get_products_by_category('shoes', 1, 20)   # Hit → instant
get_products_by_category('shoes', 2, 20)   # Miss → DB → Cached (different page)
get_products_by_category('bags', 1, 20)    # Miss → DB → Cached (different category)
```

**Memoization with time-based invalidation vs event-based:**
```python
# Time-based: cache expires after N seconds regardless of data changes
@cache.memoize(timeout=300)
def get_category_count(category):
    return db.session.query(Product).filter_by(category=category).count()

# Event-based: cache invalidated when data changes
@cache.memoize(timeout=0)  # Never auto-expire
def get_category_count(category):
    return db.session.query(Product).filter_by(category=category).count()

def add_product(category, product_data):
    db.session.add(Product(**product_data))
    db.session.commit()
    cache.delete_memoized(get_category_count, category)  # Invalidate on write
```

---

## 7. Jinja Caching (Template Caching)

**Jinja2** is Flask's templating engine. Rendering Jinja templates involves:
1. Loading the template file from disk.
2. Parsing the template syntax.
3. Executing the template with context variables.
4. Producing an HTML string.

Steps 1 and 2 can be **cached** by Jinja itself. Step 3 and 4 can be cached at the Flask level.

### 7.1 Jinja's Built-in Template Compilation Cache

Jinja2 automatically **compiles templates to Python bytecode** and caches them in memory. This means repeated rendering of the same template skips the parsing step.

```python
# Flask sets this up by default — Jinja caches compiled templates in memory
app = Flask(__name__)
# Jinja2 auto_reload is True in debug mode, False in production
# In production: template compilation is cached; no re-parsing on each render
```

**Filesystem bytecode cache (optional):**
```python
from jinja2 import FileSystemBytecodeCache

bcc = FileSystemBytecodeCache('/tmp/jinja_cache')
app.jinja_env.bytecode_cache = bcc
# Compiled templates persisted to disk — survives server restarts
```

### 7.2 Caching Rendered Template Fragments

For expensive template sections (e.g., a sidebar with complex DB data), cache the **rendered HTML fragment**:

```python
# Using Flask-Caching to cache the entire rendered template
@app.route('/products')
@cache.cached(timeout=60)
def products_page():
    products = get_products()  # expensive DB call
    categories = get_categories()  # another DB call
    return render_template('products.html',
                           products=products,
                           categories=categories)
    # The returned HTML is cached — render_template runs only on cache miss
```

**Jinja `{% cache %}` block (with `flask-caching` Jinja extension):**

For partial template caching (only specific blocks):
```python
# Register the Jinja extension
app.config['CACHE_TYPE'] = 'RedisCache'
cache = Cache(app)
cache.init_app(app, config={'JINJA2_CACHE_TYPE': 'redis'})
```

```html
<!-- In template: cache just the sidebar, not the whole page -->
{% cache 300, "sidebar" %}
  <aside class="sidebar">
    {% for category in categories %}
      <a href="/category/{{ category.slug }}">{{ category.name }}</a>
    {% endfor %}
  </aside>
{% endcache %}

<!-- The main content is dynamic (per-user), not cached -->
<main>
  <h1>Welcome, {{ current_user.name }}!</h1>
  ...
</main>
```

### 7.3 When to Cache Templates

| Scenario | Strategy |
|---|---|
| Entire page is same for all users | `@cache.cached()` on the view function |
| Page has a mix of shared + user-specific content | Cache fragments with `{% cache %}` |
| Template rarely changes | Jinja's built-in bytecode cache (automatic) |
| Template changes but data doesn't | Cache the data, not the template |

---

## 8. Caching Backends

Flask-Caching supports multiple **backends** — where the cached data is actually stored. The backend choice affects performance, scalability, and persistence.

### 8.1 NullCache

**What it is:** A "do-nothing" cache — every cache operation is a no-op. `get()` always returns `None`, `set()` does nothing.

**Configuration:**
```python
app.config['CACHE_TYPE'] = 'NullCache'
```

**Use case:**
- **Development and testing** — you want `@cache.cached()` in the code but don't want caching behaviour during testing (stale data would break tests).
- **Disabling cache** without removing cache decorators from code.
- **Feature flags** — switch to NullCache to instantly disable caching without code changes.

```python
# In test config:
app.config['CACHE_TYPE'] = 'NullCache'
# @cache.cached() decorators are ignored — view functions always execute
```

---

### 8.2 SimpleCache

**What it is:** An **in-process, in-memory** cache using a Python dictionary. Stored in the app server's memory.

**Configuration:**
```python
app.config['CACHE_TYPE'] = 'SimpleCache'
app.config['CACHE_DEFAULT_TIMEOUT'] = 300   # 5 minutes
app.config['CACHE_THRESHOLD'] = 500         # Max number of items before eviction
```

**How it works internally:**
```python
# Simplified internal representation:
_cache = {
    'view/get_products//api/products': {
        'value': b'[{"id":1,"name":"Shoes"}]',
        'expires_at': 1725362160.0
    },
    'get_user_profile:4521': {
        'value': {'id': 4521, 'name': 'Alice'},
        'expires_at': 1725362460.0
    }
}
```

**Characteristics:**
- ✅ Zero setup — no external services needed
- ✅ Fastest possible (in-process memory access)
- ❌ **NOT shared between worker processes** — each Gunicorn worker has its own cache
- ❌ **Lost on server restart**
- ❌ **Not suitable for multi-server deployments** — Server A's cache is invisible to Server B

**When to use:**
- Single-process development and simple deployments
- Caching data that can be re-computed (non-critical)
- When Redis/Memcached setup is overkill

---

### 8.3 FileSystemCache

**What it is:** Stores cached values as **files on disk**. Each cache entry is a file in a designated directory.

**Configuration:**
```python
app.config['CACHE_TYPE'] = 'FileSystemCache'
app.config['CACHE_DIR'] = '/tmp/flask_cache'   # Directory to store cache files
app.config['CACHE_DEFAULT_TIMEOUT'] = 300
app.config['CACHE_THRESHOLD'] = 1000           # Max number of files before cleanup
```

**How it works:**
```
/tmp/flask_cache/
  ├── a3f9b2c1d4e5f6a7b8c9d0e1f2a3b4c5   ← cache file for "view/get_products/..."
  ├── b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9   ← cache file for "get_user_profile:4521"
  └── c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0   ← cache file for "featured_products"
```

Each file contains a pickled Python object with the value and expiry timestamp.

**Characteristics:**
- ✅ **Survives server restarts** (data on disk)
- ✅ No external services
- ✅ Can be shared across processes on the same machine (all read/write same files)
- ❌ **Disk I/O** — slower than in-memory (SimpleCache/Redis)
- ❌ **Not suitable for multi-server deployments** — each server has its own disk
- ❌ Disk space usage — large cached objects can fill disk
- ❌ File system race conditions possible with many concurrent writes

**When to use:**
- Single-server deployments where data must survive restarts
- Caching large objects that don't fit well in Redis
- Development with a real cache but without running Redis

---

### 8.4 RedisCache

**What it is:** Uses **Redis** as the cache backend. Redis is an in-memory data structure server — it runs as a separate process (or service) and stores data in RAM.

**Configuration:**
```python
app.config['CACHE_TYPE'] = 'RedisCache'
app.config['CACHE_REDIS_URL'] = 'redis://localhost:6379/0'
# or with auth:
app.config['CACHE_REDIS_URL'] = 'redis://:password@redis-host:6379/0'
# or with Redis Cluster:
app.config['CACHE_TYPE'] = 'RedisClusterCache'
app.config['CACHE_REDIS_CLUSTER'] = [{"host": "redis1", "port": 6379}, ...]
```

**Redis data model for Flask-Caching:**
```
Redis Key-Value Store:
  Key: "flask_cache_view/get_products//api/products"
  Value: <pickled Python object>
  TTL: 60 seconds (set by EXPIRE command)

  Key: "flask_cache_get_user_profile:4521"
  Value: <pickled dict>
  TTL: 600 seconds
```

**Characteristics:**
- ✅ **Extremely fast** — in-memory, microsecond response times
- ✅ **Shared across all worker processes** on the same server
- ✅ **Shared across multiple servers** — all app servers connect to the same Redis
- ✅ **Configurable persistence** (RDB snapshots, AOF log)
- ✅ **Atomic operations** — no race conditions
- ✅ **Rich data types** (strings, hashes, lists, sets, sorted sets)
- ✅ **TTL support** built in at the protocol level
- ✅ **Production ready** — used by Twitter, GitHub, Stack Overflow
- ❌ **External service** — must be installed, configured, and maintained
- ❌ **Data must fit in RAM** — Redis stores everything in memory
- ❌ **Additional cost** in cloud environments

**When to use:**
- Any **multi-process or multi-server production deployment**
- When cache must be **shared** across all app instances
- When cache must survive a single worker restart (but can tolerate Redis restart loss)
- When you need cache entries to be **inspectable** (`redis-cli keys '*'`)

**Inspecting Flask-Caching entries in Redis CLI:**
```bash
# Connect to Redis
redis-cli

# List all Flask cache keys
KEYS flask_cache_*

# Get a cache entry's TTL
TTL flask_cache_view/get_products//api/products

# Inspect the value (will be pickled binary)
GET flask_cache_view/get_products//api/products

# Manually delete a cache entry
DEL flask_cache_get_user_profile:4521

# Flush all cache entries (equivalent to cache.clear())
FLUSHDB  ⚠️ Deletes ALL Redis keys in the database
```

---

### Backend Comparison Summary

| Backend | Speed | Shared? | Persistent? | Setup | Best For |
|---|---|---|---|---|---|
| **NullCache** | N/A | N/A | N/A | None | Testing, disabling cache |
| **SimpleCache** | Fastest | ❌ Single process | ❌ Lost on restart | None | Development, single-worker |
| **FileSystemCache** | Medium | ✅ Same machine | ✅ Survives restart | None | Single server, large objects |
| **RedisCache** | Very Fast | ✅ Any server | ✅ Configurable | Redis service | All production deployments |

---

## 9. Cache Invalidation Strategies

> **"There are only two hard things in Computer Science: cache invalidation and naming things."** — Phil Karlton

### When to Invalidate

| Strategy | Description | Example |
|---|---|---|
| **TTL (Time To Live)** | Auto-expire after N seconds | Product list cached for 60s |
| **Event-driven** | Invalidate explicitly when data changes | Delete user cache when profile updated |
| **Cache-aside** | App checks cache; on miss, loads from DB and stores | Most common pattern |
| **Write-through** | Write to cache AND DB on every write | Ensures cache is always fresh |
| **Write-behind** | Write to cache immediately, write to DB asynchronously | Very fast writes, eventual consistency |

**The safest pattern (cache-aside + event-driven invalidation):**
```python
@cache.memoize(timeout=600)
def get_product(product_id):
    return Product.query.get(product_id).to_dict()

def update_product(product_id, data):
    Product.query.filter_by(id=product_id).update(data)
    db.session.commit()
    cache.delete_memoized(get_product, product_id)  # Invalidate immediately
```

---

## 10. Key Takeaways

- **Caching = store expensive results, skip the work next time.** The core trade-off is freshness vs performance.
- Cache exists at **every layer**: browser, CDN, proxy, app, DB — each with different scope and control.
- **`Cache-Control` headers** are how the server tells every cache layer what to do with a response — learn the key directives: `max-age`, `public`/`private`, `no-store`, `no-cache`, `immutable`.
- **ETag** enables conditional requests — the browser asks "has this changed?" and gets a free 304 if it hasn't, saving bandwidth without serving stale data.
- **Freshness checking** is the browser's decision tree for whether to use, revalidate, or re-fetch a cached response.
- **Caching distorts analytics** — browser/CDN caches mean the server never sees those requests. Use client-side JS analytics (Google Analytics) for accurate counts.
- **Flask-Caching** wraps caching in clean decorators: `@cache.cached()` for views, `@cache.memoize()` for any function.
- **Cache keys must be unique per variant** — include query parameters, user ID, language, etc. in the key.
- **Memoization** caches function return values keyed on arguments — `functools.lru_cache` for in-process, `@cache.memoize()` for Redis-backed.
- **Jinja caching**: Jinja auto-caches compiled templates. Wrap entire views or fragments in `@cache.cached()` to cache rendered HTML.
- **Choose the right backend**: NullCache for tests, SimpleCache for single-process dev, FileSystemCache for single-server with persistence, **RedisCache for all production deployments**.
- **Cache invalidation is the hard part** — use TTL for low-risk data, event-driven invalidation (`delete_memoized`) for data that must be consistent after writes.
