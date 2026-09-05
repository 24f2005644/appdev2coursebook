# Speed — Detailed Notes

> **Parent Topic:** Performance
> **Scope:** Single-user experience — how fast does *one* user get a usable response?

---

## 1. What is Speed?

Speed in web development is the measure of **how quickly a web page loads and becomes usable** for a single user. It is distinct from scaling (which deals with many users simultaneously).

### Why Speed Matters

- **User attention is finite.** Research by Google shows:
  - 53% of mobile users abandon a page that takes **more than 3 seconds** to load.
  - A **1-second delay** in page load time can result in a **7% reduction** in conversions.
- **SEO impact.** Google uses Core Web Vitals (speed-related metrics) as a **direct ranking signal**.
- **First impressions.** A slow page signals low quality, even if the product itself is great.
- **Revenue.** Amazon estimated that **every 100ms of latency costs ~1% in sales**.

---

## 2. Quick Response

"Quick response" is the user-facing goal — the user clicks something and gets feedback **immediately**.

### What counts as "quick"?

Human perception has well-studied thresholds for response time:

| Delay | Perception |
|---|---|
| **< 100ms** | Feels instantaneous — no feedback needed |
| **100ms – 1s** | User notices but flow isn't disrupted; a spinner may not be needed |
| **1s – 10s** | User's attention drifts; a loading indicator is essential |
| **> 10s** | User is likely to leave or give up entirely |

### Perceived vs Actual Performance

The *feeling* of speed matters as much as raw numbers:

- **Perceived performance** = how fast the site *feels*.
- **Actual performance** = how fast it *is* (measured in ms).

Techniques to improve perceived performance (without changing actual speed):
- **Skeleton screens** — Show a placeholder layout while content loads.
- **Optimistic UI** — Apply changes in the UI before the server confirms them.
- **Progressive rendering** — Show content as it arrives rather than waiting for everything.
- **Above-the-fold priority** — Load visible content first, defer off-screen content.

---

## 3. Contributing Factors to Speed

Speed is not controlled by a single switch — it is the product of several interacting factors.

```
User clicks a link
      │
      ▼
[1] DNS Lookup          ← Network factor
      │
      ▼
[2] TCP Handshake       ← Network factor
      │
      ▼
[3] TLS Handshake       ← Network factor (HTTPS)
      │
      ▼
[4] HTTP Request sent   ← HTTP version matters
      │
      ▼
[5] Server processes    ← App-level factor
      │
      ▼
[6] Response sent back  ← Size of response + Compression
      │
      ▼
[7] Browser parses HTML ← Number of sub-requests triggered
      │
      ▼
[8] Sub-resources fetched (CSS, JS, fonts, images) ← Number of requests + Size
      │
      ▼
[9] Page rendered & interactive
```

---

### 3.1 Network

The network is the **physical and protocol layer** between the user and the server. It is often the factor developers have the *least* control over, but understanding it helps make smarter decisions.

#### Key Concepts

**Latency**
- The time for a packet to travel from the client to the server and back (round-trip time / RTT).
- Measured in milliseconds.
- Affected by: physical distance, number of network hops, congestion.
- Example: A server in the US and a user in India might have 150–200ms RTT. A CDN node in India might reduce that to 10–20ms.

**Bandwidth**
- The maximum data transfer rate of a network connection.
- Measured in Mbps or Gbps.
- High bandwidth ≠ low latency. You can have fast download speeds but high latency (e.g., satellite internet).

**Network Types and Their Impact**

| Network Type | Typical Bandwidth | Typical Latency | Notes |
|---|---|---|---|
| Broadband (Fiber) | 100–1000 Mbps | 5–20ms | Excellent for web |
| 4G LTE | 10–50 Mbps | 30–70ms | Good, but variable |
| 5G | 100–500 Mbps | 1–10ms | Best mobile option |
| 3G | 1–5 Mbps | 100–500ms | Significant bottleneck |
| Satellite | 25–100 Mbps | 500–600ms | High latency, bad for real-time |

#### What Developers Can Do
- **Use a CDN** (Content Delivery Network) to serve assets from geographically closer servers.
- **Reduce DNS lookup time** by minimizing the number of unique domains assets are served from.
- **Enable Keep-Alive** connections to reuse TCP connections across requests.
- **Preconnect** to third-party origins the browser will need:
  ```html
  <link rel="preconnect" href="https://fonts.googleapis.com">
  ```

---

### 3.2 Number of Requests

Every resource a page needs triggers a separate HTTP request. Each request incurs:
1. A DNS lookup (if new domain)
2. A TCP handshake (if new connection)
3. A TLS handshake (if HTTPS and new connection)
4. The actual request/response round trip

#### The Problem: Request Overhead

Even on a fast network, **request overhead adds up**:
- A page with 100 resources might have 100 individual round trips on HTTP/1.1.
- Even at 50ms latency each, that's 5 seconds just in overhead — before a single byte of content is counted.

#### Typical Resources That Generate Requests

```
index.html
  ├── style.css          (1 request)
  ├── reset.css          (1 request)
  ├── app.js             (1 request)
  ├── vendor.js          (1 request)
  ├── logo.png           (1 request)
  ├── hero-image.jpg     (1 request)
  ├── font.woff2         (1 request)
  ├── analytics.js       (1 request — 3rd party)
  └── /api/user          (1 request — API call)
                         = 9 requests + the HTML itself = 10 total
```

A complex real-world page can easily have **100–300 requests**.

#### Strategies to Reduce Request Count

| Strategy | How It Helps |
|---|---|
| **Bundle JS/CSS** | Combine many files into one (webpack, Vite, Rollup) |
| **CSS/JS Sprites** | Combine multiple small images into one image, use CSS to clip |
| **Inline Critical CSS** | Embed above-the-fold CSS directly in `<style>` tags |
| **Lazy Loading** | Defer loading of off-screen images/components until needed |
| **Icon Fonts / SVG Sprites** | Serve all icons in one request instead of many PNGs |
| **Reduce 3rd-party scripts** | Each external script = a new domain = DNS + TCP + TLS overhead |
| **HTTP/2 Multiplexing** | Mitigates the problem (see section 3.4) but doesn't eliminate it |

---

### 3.3 Size of Response

Larger files take longer to transfer. **Reducing the size of assets** is one of the highest-impact optimizations available.

#### Categories of Assets and Optimization Approaches

**HTML**
- Remove unnecessary whitespace, comments (minification).
- Tools: `html-minifier`, build pipeline plugins.
- Typical savings: 10–20%.

**CSS**
- **Minify:** Remove whitespace and comments.
- **Remove unused CSS:** Use PurgeCSS or built-in tree-shaking to eliminate styles not used in the HTML.
- **Avoid large frameworks** if only a fraction is used (e.g., loading all of Bootstrap for 2 components).
- Typical savings: 20–60% (minification + purging).

**JavaScript**
- **Minify:** Shorten variable names, remove whitespace (Terser, esbuild).
- **Tree-shake:** Only include code that is actually imported/used.
- **Code-split:** Don't send all JS upfront — load chunks on demand (dynamic `import()`).
- **Defer/Async:** Don't block HTML parsing.
  ```html
  <script src="app.js" defer></script>
  <script src="analytics.js" async></script>
  ```
- Typical savings: 30–70%.

**Images (often the biggest offender)**

| Format | Best For | Notes |
|---|---|---|
| JPEG | Photos | Lossy compression, small sizes |
| PNG | Transparency, diagrams | Lossless, larger than JPEG |
| **WebP** | Both | 25–35% smaller than JPEG/PNG |
| **AVIF** | Both | 50% smaller than JPEG, newer |
| SVG | Icons, logos, illustrations | Vector — scales infinitely, tiny file |
| GIF | Animations | Very inefficient — use video instead |

- **Resize images** to the display size — don't serve a 4000×3000px image for a 400×300px thumbnail.
- **Use `srcset`** to serve different sizes for different screen resolutions:
  ```html
  <img src="hero-400.jpg"
       srcset="hero-400.jpg 400w, hero-800.jpg 800w, hero-1600.jpg 1600w"
       sizes="(max-width: 600px) 400px, 800px">
  ```
- **Lazy load** images below the fold:
  ```html
  <img src="photo.jpg" loading="lazy" alt="...">
  ```

**Fonts**
- Only load font weights/styles you actually use.
- Use `font-display: swap` to avoid invisible text while font loads.
- Subset fonts to include only characters used (e.g., Latin subset vs. full Unicode).

---

### 3.4 HTTP/1 vs HTTP/2

The version of HTTP used has a **dramatic effect** on how efficiently multiple resources are fetched.

#### HTTP/1.1 — The Problem

- **One request per TCP connection** (by default).
- Browsers open **6–8 parallel connections** per domain to work around this — but this is still limited.
- **Head-of-line blocking:** If one request in a connection is slow, all subsequent requests on that connection wait.
- **No header compression:** Every request sends full headers (cookies, user-agent, etc.) repeatedly — wasteful for many small requests.

```
HTTP/1.1 timeline (simplified):

Connection 1:  [request1]→[response1]  [request4]→[response4]
Connection 2:  [request2]→[response2]  [request5]→[response5]
Connection 3:  [request3]→[response3]  [request6]→[response6]
               ← sequential within each connection →
```

#### HTTP/2 — The Solution

- **Multiplexing:** Multiple requests and responses are sent **simultaneously over a single TCP connection**.
- **Header compression (HPACK):** Headers are compressed and deduplicated across requests.
- **Stream prioritization:** The server can prioritize which resources to send first.
- **Server Push:** The server can proactively send resources the client will need before it asks.

```
HTTP/2 timeline (simplified):

Single Connection:
  ─── [req1] [req2] [req3] [req4] [req5] [req6] ──▶
  ◀── [res3] [res1] [res5] [res2] [res6] [res4] ───
       (responses can arrive out of order — all simultaneous)
```

#### HTTP/3 (Bonus — Modern Context)
- Built on **QUIC** (UDP-based) instead of TCP.
- Eliminates TCP-level head-of-line blocking.
- Faster connection establishment (0-RTT or 1-RTT).
- Better performance on lossy networks (mobile).

#### Comparison Table

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | ❌ No | ✅ Yes | ✅ Yes |
| Header Compression | ❌ No | ✅ HPACK | ✅ QPACK |
| Server Push | ❌ No | ✅ Yes | ✅ Yes |
| Head-of-line Blocking | ❌ TCP + App level | ❌ TCP level only | ✅ Eliminated |
| Connection Setup | Slow (TCP + TLS) | Slow (TCP + TLS) | Fast (0-RTT) |

#### Developer Implications
- Most modern hosting (Nginx, Apache, cloud providers) supports HTTP/2 — **enable it**.
- HTTP/2 makes domain sharding (splitting assets across domains) **counterproductive** — it hurts header compression.
- Test what version your server uses: Chrome DevTools → Network tab → Protocol column.

---

### 3.5 Compression

Compression reduces the **number of bytes transferred** by encoding the response more efficiently. The browser decompresses it before use — this is transparent to the user.

#### How It Works

```
Server                          Browser
  │                                │
  │  ← GET /app.js ─────────────── │
  │    Accept-Encoding: gzip, br   │
  │                                │
  │  ─── HTTP 200 ───────────────► │
  │    Content-Encoding: br        │
  │    [compressed body: 23KB]     │  ← Browser decompresses
  │                                │    Original: 120KB
```

#### Compression Algorithms

**gzip**
- Available since the mid-1990s.
- Universally supported by all browsers and servers.
- Typically achieves **60–70% compression** on text content.
- Default choice when broad compatibility is needed.

**Brotli (br)**
- Developed by Google, released in 2015.
- **10–25% better compression** than gzip for the same quality.
- Supported by all modern browsers (Chrome, Firefox, Safari, Edge).
- Slightly more CPU-intensive to compress (but decompression is fast).
- **Recommended for production** — most CDNs support it.

**Comparison**

| Algorithm | Compression Ratio | Browser Support | CPU to Compress |
|---|---|---|---|
| **gzip** | Good (~65%) | Universal | Low |
| **Brotli** | Better (~75%) | Modern browsers | Medium |
| **zstd** | Best (emerging) | Limited | Low |

#### What Compresses Well vs. Poorly

| Asset Type | Compresses Well? | Notes |
|---|---|---|
| HTML | ✅ Yes | Text is highly compressible |
| CSS | ✅ Yes | Lots of repetitive patterns |
| JavaScript | ✅ Yes | Even after minification |
| JSON / XML | ✅ Yes | Very repetitive structures |
| SVG | ✅ Yes | It's XML |
| JPEG / PNG / WebP | ❌ No | Already compressed internally |
| MP4 / WebM | ❌ No | Already compressed |
| ZIP / GZ files | ❌ No | Already compressed |

#### Enabling Compression

**Nginx:**
```nginx
gzip on;
gzip_types text/plain text/css application/javascript application/json;

# For Brotli (requires ngx_brotli module):
brotli on;
brotli_types text/plain text/css application/javascript application/json;
```

**Apache:**
```apache
# In .htaccess or httpd.conf:
AddOutputFilterByType DEFLATE text/html text/css application/javascript
```

**Express.js (Node):**
```javascript
const compression = require('compression');
app.use(compression()); // automatically compresses responses
```

**Flask (Python):**
```python
from flask_compress import Compress
Compress(app)  # automatically compresses responses
```

---

## 4. How These Factors Interact

Real-world speed is a product of **all these factors simultaneously**:

```
Total Load Time ≈
    DNS Lookup Time
  + TCP Handshake (× number of new connections)
  + TLS Handshake (× number of new connections)
  + (Request + Response time × number of requests) / HTTP2 parallelism
  + (Response bytes / bandwidth) after compression
  + Browser parse & render time
```

**Example optimization cascade:**

| Step | Before | After | Saving |
|---|---|---|---|
| Enable HTTP/2 | 100 sequential requests | 100 parallel requests | ~70% time reduction |
| Enable Brotli | 500KB JS | 125KB JS | 75% bytes saved |
| Lazy-load images | 50 images at load | 5 images at load | 45 fewer requests |
| Use WebP | 2MB images | 1.3MB images | 35% bytes saved |
| Bundle JS | 40 JS files | 2 JS files | 38 fewer requests |

---

## 5. Key Takeaways

- **Speed is a single-user concern** — how fast does one request-response cycle feel?
- The **100ms rule**: responses under 100ms feel instant; over 1s, users feel the wait.
- **Perceived performance** can be improved independently of actual performance using skeleton screens and optimistic UI.
- The **5 main contributing factors** are: Network, Number of Requests, Size of Response, HTTP version, and Compression.
- **HTTP/2 multiplexing** is the single biggest protocol-level improvement — enable it on your server.
- **Compression (Brotli preferred)** can cut text asset sizes by 60–75% for free.
- **Images** are often the largest contributors to page weight — use modern formats (WebP/AVIF), resize correctly, and lazy-load.
- Always **measure with Lighthouse and DevTools Network tab** before and after optimization.
