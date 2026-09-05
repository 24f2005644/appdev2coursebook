# Performance — Detailed Notes

---

## 1. Overview

**Performance** in web development refers to how fast and efficiently a web application responds to user interactions. It is broadly split into two dimensions:

| Dimension | Focus | Concerned With |
|---|---|---|
| **Speed** | Single user | How fast does one user get a response? |
| **Scaling** | Multiple users | How does the app behave under load? |

> Think of speed as the *quality* of a single experience, and scaling as *maintaining* that quality for thousands of concurrent users.

---

## 2. Speed

### 2.1 What is Speed?

Speed refers to the **time it takes for a server to respond** to a single user's request and for that content to be rendered in the browser.

- A fast site feels **snappy and responsive**.
- A slow site leads to **user drop-off** — studies show that even a 1-second delay can reduce conversions by ~7%.
- Speed is a **single-user concern** — it doesn't care about how many people are using the site at the same time.

---

### 2.2 Contributing Factors to Speed

#### 🌐 Network
- The physical medium and protocol used to transmit data between the server and the user.
- Latency (round-trip time) and bandwidth both affect speed.
- Mobile networks (4G/5G) introduce higher and more variable latency compared to broadband.

#### 📦 Number of Requests
- Every resource (HTML, CSS, JS, image, font, API call) requires a **separate HTTP request**.
- More requests = more round trips = slower page load.
- **Optimization:** Bundle files, use sprites, inline critical CSS, lazy-load assets.

#### 📐 Size of Response
- Larger files take longer to download.
- **Optimization strategies:**
  - Minify CSS, JS, and HTML (remove whitespace, comments).
  - Compress images (use modern formats like WebP, AVIF).
  - Tree-shake unused JavaScript.

#### ⚡ HTTP/1 vs HTTP/2

| Feature | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Connection | One request per TCP connection | Multiplexed (multiple requests over one connection) |
| Header Compression | None | HPACK compression |
| Server Push | Not supported | Supported |
| Performance | Slower (head-of-line blocking) | Significantly faster |

- **HTTP/2** dramatically reduces the impact of multiple requests by multiplexing them over a single connection.
- Most modern servers and browsers support HTTP/2.

#### 🗜️ Compression
- Servers can compress responses before sending them, and browsers decompress them.
- Common algorithms:
  - **gzip** — widely supported, good compression ratio.
  - **Brotli** — newer, better compression than gzip, supported by modern browsers.
- Enabled via HTTP headers:
  ```
  Content-Encoding: gzip
  Accept-Encoding: gzip, deflate, br
  ```
- Text-based resources (HTML, CSS, JS, JSON) compress very well (often 60–80% size reduction).
- Already-compressed formats (JPEG, PNG, MP4) see minimal benefit.

---

## 3. User Experience (UX)

### 3.1 UI vs UX — What's the Difference?

| Term | Stands For | Focus |
|---|---|---|
| **UI** | User Interface | The *look* — visual design, layout, colors, typography |
| **UX** | User Experience | The *feel* — how smooth, intuitive, and satisfying the interaction is |

- A site can have a beautiful UI but poor UX (e.g., slow load times, jarring layout shifts).
- **Performance is a core part of UX.** A slow app is a bad experience, regardless of how it looks.

### 3.2 Why UX Matters for Performance
- Users perceive performance subjectively. A page that *feels* fast (even if it isn't fully loaded) is better UX.
- Techniques like **skeleton screens**, **optimistic UI**, and **progressive loading** improve perceived performance.
- **Core Web Vitals** (Google's metrics) directly tie performance to search ranking, connecting UX to business outcomes.

---

## 4. Tools & Measurement

> **Golden Rule: Measure first, then optimize.** Never guess where bottlenecks are.

### 4.1 How to Measure Performance

Performance can be measured at different levels:
- **Lab testing** — Controlled environment, simulated conditions (e.g., Lighthouse, WebPageTest).
- **Real User Monitoring (RUM)** — Actual user data collected in production (e.g., Google Analytics, Sentry).

---

### 4.2 Google Lighthouse

**Lighthouse** is an open-source, automated tool for auditing web page quality, built into Chrome DevTools.

#### Architecture
```
Browser (Chrome DevTools / CLI / CI)
        │
        ▼
   Lighthouse Runner
        │
        ├── Loads the page in a controlled environment
        ├── Simulates network conditions (e.g., Slow 4G)
        ├── Simulates CPU throttling
        └── Collects trace & network data
        │
        ▼
   Audit Engine
        │
        ├── Performance audits
        ├── Accessibility audits
        ├── Best Practices audits
        └── SEO audits
        │
        ▼
   Report Generation (Score 0–100 per category)
```

#### What Does Lighthouse Do?
- Runs a series of **audits** against a page.
- Simulates **real-world conditions** (throttled network, throttled CPU).
- Produces a **score from 0–100** for each category.
- Provides **actionable recommendations** with estimated savings.
- Can be run via:
  - Chrome DevTools (F12 → Lighthouse tab)
  - CLI: `npx lighthouse <url>`
  - CI/CD pipelines for automated regression testing

---

### 4.3 Performance Metrics (Core Web Vitals + More)

These are the specific metrics Lighthouse (and browsers) use to quantify page speed and user experience.

#### 🎨 First Contentful Paint (FCP)
- **What:** Time from navigation start until the browser renders the **first piece of content** (text, image, SVG, canvas).
- **Why it matters:** Signals to the user that the page is *actually loading* — reassures them the request worked.
- **Good score:** < 1.8 seconds

#### 📏 Speed Index (SI)
- **What:** Measures how **quickly the visual content of a page is populated** — it's an average of how fast each part of the viewport becomes visible.
- **Why it matters:** A low Speed Index means users see content sooner, even if the full page isn't loaded.
- **Good score:** < 3.4 seconds

#### 🖼️ Largest Contentful Paint (LCP)
- **What:** Time from navigation start until the **largest visible content element** (image, video, block-level text) is rendered.
- **Why it matters:** Tracks when the **main content** is loaded — the most important piece of content the user came to see.
- **Good score:** < 2.5 seconds
- **Core Web Vital** ✅

#### 🖱️ Time to Interactive (TTI)
- **What:** Time from navigation start until the page is **fully interactive** — all event handlers registered, responds to user input within 50ms.
- **Why it matters:** A page might *look* loaded but be unresponsive (JS still parsing/executing). TTI captures this gap.
- **Good score:** < 3.8 seconds

#### ⏱️ Total Blocking Time (TBT)
- **What:** Total time between FCP and TTI where the **main thread is blocked** for more than 50ms (by long tasks).
- **Why it matters:** Long blocking tasks make the page feel sluggish — the browser can't respond to user input during them.
- **Good score:** < 200 milliseconds
- **Core Web Vital** ✅ (lab proxy for Interaction to Next Paint)

#### 📐 Cumulative Layout Shift (CLS)
- **What:** Measures the **visual stability** of a page — how much elements *unexpectedly shift* during loading.
- **Why it matters:** Annoying UX issue — you go to click a button and it jumps away. Common causes: images without dimensions, dynamically injected content, web fonts loading.
- **Good score:** < 0.1
- **Core Web Vital** ✅

##### Summary Table

| Metric | Measures | Good Threshold | Core Web Vital? |
|---|---|---|---|
| FCP | First content visible | < 1.8s | No |
| Speed Index | Visual completeness speed | < 3.4s | No |
| LCP | Main content loaded | < 2.5s | ✅ Yes |
| TTI | Fully interactive | < 3.8s | No |
| TBT | Main thread blocking | < 200ms | No (proxy for INP) |
| CLS | Layout stability | < 0.1 | ✅ Yes |

---

### 4.4 Other Parameters Lighthouse Audits

Beyond raw performance, Lighthouse also scores:

#### ♿ Accessibility
- Checks whether the site is usable by people with disabilities.
- Examples: alt text on images, proper ARIA labels, sufficient color contrast, keyboard navigability.
- Score reflects how well the site conforms to **WCAG guidelines**.

#### ✅ Best Practices
- Checks for general web hygiene:
  - HTTPS usage
  - No deprecated APIs
  - No browser errors in the console
  - Correct image aspect ratios
  - Avoiding document.write()

#### 🔍 Search Engine Optimization (SEO)
- Checks for basics that help search engines crawl and index the page:
  - Presence of `<title>` and `<meta description>` tags
  - Legible font sizes
  - Tap targets sized appropriately
  - `robots.txt` and crawlability

---

### 4.5 Problems & Limitations of Automated Checks

Lighthouse and similar tools are **valuable but not perfect**. Key limitations:

| Limitation | Explanation |
|---|---|
| **Lab ≠ Real World** | Scores are from a simulated environment. Real users have different devices, networks, and locations. |
| **Single snapshot** | A single run captures one moment. Performance can vary due to server load, CDN routing, etc. |
| **No user behavior** | Can't measure how users *actually* interact with the page (scroll depth, rage clicks, etc.). |
| **False positives/negatives** | A high score doesn't guarantee a good experience; a low score doesn't always mean users notice. |
| **Doesn't test all pages** | Typically run on one URL — deeply nested pages or pages behind auth may be missed. |
| **Gaming the score** | Developers can optimize specifically for Lighthouse metrics without improving real UX. |
| **Dynamic content** | Sites that load data after initial paint (SPAs, dashboards) may score poorly despite fast real-world UX. |

> **Bottom line:** Use Lighthouse as a **starting point and regression detector**, not as the sole measure of performance. Complement it with **Real User Monitoring (RUM)** and actual user feedback.

---

## 5. Key Takeaways

- **Performance = Speed + Scaling.** Speed is single-user; scaling is multi-user.
- Speed is impacted by **network, request count, response size, HTTP version, and compression**.
- **UX is inseparable from performance** — slow = bad experience.
- **Lighthouse** is the go-to tool for measuring performance in a controlled environment; understand its architecture and what each metric means.
- The **6 core metrics** (FCP, SI, LCP, TTI, TBT, CLS) each capture a different dimension of the loading experience.
- **Automated tools have limits** — always combine them with real-world monitoring.
- **Measure first, then optimize.** Never prematurely optimize without data.
