# Tools & Measurement — Detailed Notes

> **Parent Topic:** Performance
> **Scope:** How to objectively measure web performance, what tools exist, what each metric means, and where automated tools fall short.

---

## 1. How to Measure Performance

> **Golden Rule: Measure first, then optimize. Never guess.**

Without measurement, optimization is guesswork — you might spend days improving something that doesn't affect the user experience at all, while a real bottleneck goes unnoticed.

### Two Environments for Measurement

| Environment | Type | Description | Examples |
|---|---|---|---|
| **Lab (Synthetic)** | Controlled | Simulated, reproducible conditions | Lighthouse, WebPageTest |
| **Field (Real User)** | Real-world | Actual user data from production | CrUX, Google Analytics, Sentry |

Each has trade-offs:

```
Lab Testing                         Field (RUM) Testing
─────────────────────────────       ─────────────────────────────
✅ Reproducible                     ✅ Real-world accuracy
✅ Diagnosable (trace data)         ✅ Captures all device/network types
✅ Works before launch              ✅ Captures real user frustration
✅ Automatable in CI/CD             ✅ Statistical significance over time
❌ Simulated conditions             ❌ Only works post-launch
❌ Single device/config             ❌ Harder to debug specific issues
❌ May not reflect real users       ❌ Slower feedback loop
```

> **Best practice:** Use **both**. Lab for development feedback; RUM for production truth.

### The Measurement Lifecycle

```
Development          Staging / Pre-prod          Production
─────────────        ──────────────────          ──────────────────
Run Lighthouse       Run WebPageTest             Collect RUM data
in DevTools          against realistic            (CrUX, Analytics)
                     server environment
      │                      │                          │
      ▼                      ▼                          ▼
 Fast feedback          Catch regressions          Real truth
 (seconds)             before users do            (ongoing)
```

---

## 2. Google Lighthouse

**Lighthouse** is the most widely used web performance auditing tool. It is open-source, maintained by Google, and built directly into Chrome DevTools.

### What is Lighthouse?

- An automated, auditing tool that tests a single URL for **performance, accessibility, best practices, and SEO**.
- Produces a **score from 0–100** for each category.
- Gives **specific, actionable recommendations** with estimated savings (e.g., "Serve images in next-gen formats: save 0.45s").
- Available in multiple environments:
  - **Chrome DevTools** — F12 → "Lighthouse" tab → Generate report
  - **CLI** — `npx lighthouse https://example.com --output html`
  - **CI/CD** — via `lighthouse-ci` npm package to block deploys on regressions
  - **PageSpeed Insights** — `https://pagespeed.web.dev` (Google's hosted version)

---

### 2.1 Lighthouse Architecture

Understanding how Lighthouse works helps you interpret its results correctly.

```
┌────────────────────────────────────────────────────────────────┐
│                        LIGHTHOUSE RUNNER                       │
│                                                                │
│  1. SETUP                                                      │
│     ├── Launches Chrome in headless mode                       │
│     ├── Applies throttling:                                    │
│     │     ├── Network: Simulated Slow 4G (40ms RTT, 10Mbps)   │
│     │     └── CPU: 4× slowdown (simulates mid-range mobile)   │
│     └── Clears cache, cookies (fresh load simulation)         │
│                                                                │
│  2. PAGE LOAD                                                  │
│     ├── Navigates to the target URL                            │
│     ├── Collects a Chrome performance trace                    │
│     ├── Intercepts all network requests                        │
│     └── Waits for the page to settle (network idle)           │
│                                                                │
│  3. AUDIT ENGINE                                               │
│     ├── Runs 50+ individual audits against the trace/DOM      │
│     ├── Each audit is pass/fail or numeric                     │
│     └── Audits grouped into 5 categories                      │
│                                                                │
│  4. SCORING                                                    │
│     ├── Each metric scored against a log-normal distribution  │
│     ├── Metrics weighted and combined into category score     │
│     └── Score: 0–49 (Poor) | 50–89 (Needs Work) | 90–100 (Good)│
│                                                                │
│  5. REPORT                                                     │
│     ├── HTML / JSON / CSV output                               │
│     ├── Opportunities (estimated savings)                      │
│     └── Diagnostics (informational findings)                  │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 What Does Lighthouse Do?

Lighthouse runs **two types of evaluations** per category:

**Audits** — Specific checks, each with a pass/fail or scored result:
- "Does every `<img>` have an `alt` attribute?" → Pass/Fail
- "What is the LCP time?" → 1.8s → Score: 92

**Opportunities** — Actionable suggestions with estimated time savings:
- "Eliminate render-blocking resources → potential savings: 0.9s"
- "Properly size images → potential savings: 1.2s"

**Diagnostics** — Informational (not directly scored):
- "Keep request counts low and transfer sizes small"
- "Minimize main-thread work"

### How the Performance Score is Calculated

The Performance score is a **weighted average** of 6 metrics:

| Metric | Weight |
|---|---|
| Largest Contentful Paint (LCP) | 25% |
| Total Blocking Time (TBT) | 30% |
| Cumulative Layout Shift (CLS) | 15% |
| First Contentful Paint (FCP) | 10% |
| Speed Index (SI) | 10% |
| Time to Interactive (TTI) | 10% |

> Note: Weights change between Lighthouse versions. Always check the current version's weighting.

---

## 3. Performance Metrics

These 6 metrics are the core of what Lighthouse measures for performance. Each captures a different dimension of the loading experience.

---

### 3.1 First Contentful Paint (FCP)

**What it measures:**
The time from when the user navigates to the page until the browser renders the **first piece of DOM content** — any text, image, SVG, or non-white `<canvas>` element.

**Why it matters:**
- Answers: *"Is anything happening?"*
- The first signal to the user that the server responded and the page is loading.
- Without FCP, the user sees a blank white screen and doesn't know if the page is broken or loading.

**Timeline:**
```
Navigation Start ──────────────────────────────────────────────►
     0ms          |FCP|        |LCP|           |TTI|
                  ↑
                  First text/image visible
```

**Scoring Thresholds:**

| Score | FCP Time |
|---|---|
| 🟢 Good | 0 – 1.8s |
| 🟡 Needs Improvement | 1.8s – 3.0s |
| 🔴 Poor | > 3.0s |

**Common causes of slow FCP:**
- Render-blocking CSS or JS (browser must download and parse these before rendering)
- Large HTML document (server is slow to generate or send)
- No server-side rendering (blank HTML shell waiting for JS to populate)
- Slow server response (TTFB — Time to First Byte — is high)

**How to improve:**
- Inline critical CSS
- Defer non-critical JS (`defer` / `async`)
- Use a CDN to reduce TTFB
- Enable server-side rendering (SSR)

---

### 3.2 Speed Index (SI)

**What it measures:**
A composite metric that measures **how quickly the visual content of a page is populated** over time. It considers the entire loading process, not just the first or last moment of content appearing.

Technically: It captures video frames of the page loading and calculates the average time at which each pixel in the viewport becomes "visually complete."

**Why it matters:**
- Answers: *"How quickly is the page filling up with content?"*
- A page might have a good FCP (one word appears quickly) but a bad Speed Index (everything else takes ages).
- Reflects the overall *pace* of visual loading.

**Analogy:**
```
Page A: [▓░░░░░░░░░] → [▓▓░░░░░░░] → [▓▓▓▓▓▓▓▓▓]   ← Slow SI (gradual fill)
Page B: [▓▓▓▓░░░░░] → [▓▓▓▓▓▓▓░░] → [▓▓▓▓▓▓▓▓▓]   ← Fast SI (quick fill)
        t=0.5s          t=1.5s          t=2.5s
```

**Scoring Thresholds:**

| Score | Speed Index |
|---|---|
| 🟢 Good | 0 – 3.4s |
| 🟡 Needs Improvement | 3.4s – 5.8s |
| 🔴 Poor | > 5.8s |

**How to improve:**
- Minimize render-blocking resources
- Optimize and compress images
- Reduce JavaScript execution time

---

### 3.3 Largest Contentful Paint (LCP)

**What it measures:**
The time from navigation start until the **largest visible content element** in the viewport is fully rendered. Tracked elements: `<img>`, `<image>` in SVG, `<video>` (poster frame), elements with a CSS `background-image`, and large block-level text nodes.

**Why it matters:**
- Answers: *"When does the main content load?"*
- This is the most important metric for perceived loading speed — it marks when the **hero content** the user came to see is visible.
- **Core Web Vital** — used directly by Google as a search ranking signal.

**Scoring Thresholds:**

| Score | LCP Time |
|---|---|
| 🟢 Good | 0 – 2.5s |
| 🟡 Needs Improvement | 2.5s – 4.0s |
| 🔴 Poor | > 4.0s |

**Common LCP elements:**
- Hero images
- Large heading text (`<h1>`)
- Video thumbnails / poster images
- Full-width banner images

**Common causes of slow LCP:**
- Large, unoptimized hero image (biggest offender)
- Image not preloaded
- Render-blocking JS/CSS delaying the element
- Slow server response (high TTFB)
- Client-side rendering (image loaded via JS, not HTML)

**How to improve:**
```html
<!-- Preload the LCP image -->
<link rel="preload" as="image" href="hero.webp">

<!-- Use modern image format -->
<img src="hero.webp" alt="Hero" fetchpriority="high">

<!-- Give images explicit dimensions to avoid CLS -->
<img src="hero.webp" width="1200" height="600" alt="Hero">
```

---

### 3.4 Time to Interactive (TTI)

**What it measures:**
The time from navigation start until the page is **reliably interactive** — defined as:
1. The page has displayed useful content (past FCP).
2. Event handlers are registered for most visible elements.
3. The page responds to user interactions within **50ms**.

**Why it matters:**
- Answers: *"When can I actually use this page?"*
- A page can *look* fully loaded while the browser is still parsing and executing JavaScript, making it unresponsive.
- Captures the frustrating experience of clicking a button that does nothing.

**The Gap Between Visual Load and Interactivity:**
```
Navigation ──────────────────────────────────────────────────►
           |FCP|          |LCP|         |TTI|
                    ↑                   ↑
           Page looks loaded      Page actually works
           (visual)               (interactive)
                    └── "Zombie Phase" ──┘
                        (looks alive, but can't interact)
```

**Scoring Thresholds:**

| Score | TTI |
|---|---|
| 🟢 Good | 0 – 3.8s |
| 🟡 Needs Improvement | 3.8s – 7.3s |
| 🔴 Poor | > 7.3s |

**Common causes of high TTI:**
- Large JavaScript bundles that take long to parse and execute
- Synchronous third-party scripts (ads, analytics, chatbots)
- Long tasks on the main thread

**How to improve:**
- Code-split JavaScript — load only what's needed for the current page
- Move non-critical JS to web workers
- Defer third-party scripts
- Use `<script defer>` / `<script async>`

---

### 3.5 Total Blocking Time (TBT)

**What it measures:**
The **total time** between FCP and TTI during which the main thread was blocked for **more than 50ms** at a time (by "long tasks"). Only the portion *over* 50ms is counted.

**Why it matters:**
- Answers: *"How much of the loading period was the page unresponsive to input?"*
- The browser's main thread handles both JavaScript execution and user input. When a long task runs, **user input is queued** until the task completes.
- A high TBT means the page regularly felt frozen during loading.
- **Core Web Vital** (lab proxy) — correlates strongly with real-world Interaction to Next Paint (INP).

**How Long Tasks are Counted:**
```
Main Thread ──────────────────────────────────────────────────►

Task A: [░░░░░░░░░░░░░░░░░░░░░] = 200ms total
        [──────50ms────][░░░░░] → Blocking portion = 150ms

Task B: [░░░░░░░░░] = 80ms total
        [──50ms──][░░] → Blocking portion = 30ms

Task C: [░░░░] = 30ms → Not a long task → Blocking = 0ms

TBT = 150ms + 30ms = 180ms
```

**Scoring Thresholds:**

| Score | TBT |
|---|---|
| 🟢 Good | 0 – 200ms |
| 🟡 Needs Improvement | 200ms – 600ms |
| 🔴 Poor | > 600ms |

**How to improve:**
- Break up long JavaScript tasks into smaller chunks (using `setTimeout`, `requestIdleCallback`, or scheduler API)
- Reduce the amount of JavaScript executed during page load
- Remove or defer unused third-party scripts

---

### 3.6 Cumulative Layout Shift (CLS)

**What it measures:**
A measure of **visual stability** — how much the page layout unexpectedly shifts during the entire lifetime of the page. Calculated as:

```
CLS = Σ (layout_shift_score) for all unexpected shifts
    = Σ (impact_fraction × distance_fraction)

Where:
  impact_fraction = % of viewport affected by the shift
  distance_fraction = % of viewport the element moved
```

**Why it matters:**
- Answers: *"Does the page stay stable as it loads?"*
- Layout shifts are one of the most frustrating UX issues — you're about to click something and it jumps away.
- **Core Web Vital** — used directly by Google as a ranking signal.

**Visualized:**
```
Before shift:                    After shift:
┌──────────────────────┐         ┌──────────────────────┐
│  [Article Title]     │         │  [AD BANNER]         │  ← inserted
│                      │    →    │  [Article Title]     │  ← shifted down
│  [Read more]  ←──────┼─────────┼──[Read more]         │
│                      │  User   │                      │
│  [AD BANNER]         │  clicks │                      │
└──────────────────────┘  here!  └──────────────────────┘
```

**Scoring Thresholds:**

| Score | CLS |
|---|---|
| 🟢 Good | 0 – 0.1 |
| 🟡 Needs Improvement | 0.1 – 0.25 |
| 🔴 Poor | > 0.25 |

**Common causes:**
- Images / videos without explicit `width` and `height` attributes
- Ads, embeds, or iframes with unknown dimensions
- Web fonts causing text to reflow when they load (`FOUT` — Flash of Unstyled Text)
- Content dynamically injected above existing content

**How to fix:**
```html
<!-- Always specify dimensions on images -->
<img src="photo.jpg" width="800" height="600" alt="...">

<!-- Reserve space for dynamic content with CSS -->
.ad-container {
  min-height: 250px; /* reserve space before ad loads */
}

/* Avoid FOUT with font-display */
@font-face {
  font-family: 'MyFont';
  font-display: optional; /* or 'swap' */
}
```

---

### Metrics Summary Table

| Metric | Measures | Good | Needs Work | Poor | Core Web Vital? | Weight |
|---|---|---|---|---|---|---|
| **FCP** | First content appears | < 1.8s | 1.8–3.0s | > 3.0s | No | 10% |
| **Speed Index** | Pace of visual loading | < 3.4s | 3.4–5.8s | > 5.8s | No | 10% |
| **LCP** | Main content loaded | < 2.5s | 2.5–4.0s | > 4.0s | ✅ Yes | 25% |
| **TTI** | Fully interactive | < 3.8s | 3.8–7.3s | > 7.3s | No | 10% |
| **TBT** | Main thread blocked | < 200ms | 200–600ms | > 600ms | Proxy | 30% |
| **CLS** | Layout stability | < 0.1 | 0.1–0.25 | > 0.25 | ✅ Yes | 15% |

---

## 4. Other Parameters Lighthouse Audits

Beyond the performance score, Lighthouse audits three other categories, each producing its own 0–100 score.

---

### 4.1 Accessibility

**What it checks:**
Whether the site is usable by people with disabilities — visual, motor, auditory, or cognitive.

**Key audits:**
| Check | Example |
|---|---|
| Image alt text | Every `<img>` must have `alt` attribute |
| Color contrast | Text must have sufficient contrast ratio (4.5:1 for normal, 3:1 for large text) |
| ARIA labels | Interactive elements must have accessible names |
| Form labels | Every `<input>` must have an associated `<label>` |
| Keyboard navigation | All interactive elements reachable and usable via keyboard |
| Heading order | Headings must follow a logical hierarchy (h1 → h2 → h3) |
| Language attribute | `<html lang="en">` must be set |

**Standards it checks against:** WCAG 2.1 (Web Content Accessibility Guidelines)

> ⚠️ **Caveat:** Lighthouse catches ~30% of accessibility issues. Many require human judgment (e.g., does the alt text actually describe the image meaningfully?).

---

### 4.2 Best Practices

**What it checks:**
General web hygiene — secure, modern, and correct usage of browser APIs.

**Key audits:**
| Check | Why It Matters |
|---|---|
| HTTPS | Prevents man-in-the-middle attacks; required for many browser APIs |
| No deprecated APIs | `document.write()`, `XMLHttpRequest` replacements, etc. |
| No browser console errors | Errors indicate broken functionality |
| Correct image aspect ratios | Distorted images signal low quality |
| `<DOCTYPE>` present | Ensures browser uses standards mode |
| No mixed content | HTTP resources on HTTPS pages compromise security |
| Avoid `eval()` | Security and performance risk |
| `rel="noopener"` on `target="_blank"` links | Prevents tab-napping attacks |

---

### 4.3 Search Engine Optimization (SEO)

**What it checks:**
Basic signals that help search engines discover, crawl, and understand the page.

**Key audits:**

| Check | What to Do |
|---|---|
| `<title>` tag | Every page needs a unique, descriptive title |
| `<meta name="description">` | Summarize page content (150–160 characters) |
| Legible font sizes | Body text should be ≥ 12px |
| Tap target sizing | Buttons/links ≥ 48×48px on mobile |
| `robots.txt` | Should not block crawler access to important resources |
| Crawlable links | Links must use `<a href>`, not JS-only click handlers |
| `hreflang` | Correct language/region targeting for international sites |
| Structured data | Valid schema.org markup (eligible for rich results) |

> ⚠️ **Caveat:** Lighthouse's SEO score only checks technical basics. It cannot assess content quality, backlinks, keyword relevance, or domain authority — which are far more important ranking factors.

---

## 5. Problems & Limitations of Automated Checks

Lighthouse is a powerful tool, but it has **real limitations** that developers must understand to avoid misusing the results.

### 5.1 Lab ≠ Real World

Lighthouse simulates one specific device + network condition (mid-range Android, Slow 4G). Real users have:
- Different devices (low-end phones to high-end desktops)
- Different network conditions (2G to fiber)
- Different geographic locations (latency varies)
- Different browsers (Chrome, Safari, Firefox, Edge)

A score of 95 in Lighthouse might correspond to a score of 60 in the field for users on older devices in rural areas.

### 5.2 Single Point-in-Time Measurement

- A Lighthouse run is a **snapshot** — one run, one moment.
- Scores can vary significantly between runs (±10–15 points) due to:
  - Server-side variability
  - Background tasks on the test machine
  - Randomness in Chrome's internal processes
- **Solution:** Run Lighthouse 3–5 times and use the median score. Use Lighthouse CI for consistent automated baselines.

### 5.3 Doesn't Capture Dynamic Behavior

Lighthouse tests the **initial page load only**. It does not capture:
- Performance of Single Page Application (SPA) route transitions
- Performance of user interactions after load (clicking, scrolling, filtering)
- Performance of features behind authentication (dashboards, admin panels)
- Long-session performance (memory leaks, accumulated slowdowns)

### 5.4 Score Gaming

Developers can achieve a high Lighthouse score without delivering a genuinely good user experience:
- Lazy-loading everything (good for score, bad if users never see content)
- Deferring all JS (score improves, but app is unusable until JS loads)
- Running Lighthouse on a page with no real content (empty state scores well)

### 5.5 No User Behavior Data

Lighthouse cannot tell you:
- Where users are rage-clicking (clicking repeatedly because something isn't working)
- Where users are getting confused and abandoning tasks
- Which pages have the most real-world performance problems
- What the 95th percentile experience looks like (outliers who have it worst)

### 5.6 Accessibility Gaps

- Lighthouse catches only **~30% of accessibility issues** automatically.
- Many issues require human judgment: meaningful alt text, logical reading order, cognitive clarity.
- A score of 100 in accessibility does **not** mean the site is fully accessible.

### 5.7 Does Not Test All Pages

- Typically run on one or a few URLs.
- Large sites have hundreds or thousands of pages — Lighthouse won't find performance issues on product detail pages, checkout flows, or paginated lists.

---

### Summary of Limitations

| Limitation | Mitigation |
|---|---|
| Simulated conditions | Supplement with RUM (Chrome UX Report, Sentry) |
| Score variability | Run multiple times; use median; use Lighthouse CI |
| SPA / post-load performance | Use browser Profiler, trace analysis, Web Vitals library |
| Auth-gated pages | Run Lighthouse via CI with authenticated sessions |
| Score gaming | Focus on real user metrics, not just the Lighthouse score |
| Accessibility gaps | Combine with manual audits and screen reader testing |
| Single page only | Integrate Lighthouse CI across multiple URLs |

---

## 6. Key Takeaways

- **Always measure before optimizing** — guessing leads to wasted effort.
- Use **both lab (Lighthouse) and field (RUM) data** for a complete picture.
- **Lighthouse architecture:** headless Chrome + throttling + 50+ audits → 0–100 category scores.
- Know the **6 performance metrics**, what each measures, and what "good" looks like.
- **LCP, TBT, and CLS** carry the most weight in the Lighthouse Performance score (70% combined).
- **LCP and CLS are Core Web Vitals** — they directly influence Google Search ranking.
- Lighthouse also audits **Accessibility, Best Practices, and SEO** — treat all four categories seriously.
- **Automated checks have real limits** — a high score doesn't guarantee a good experience. Use them as starting points, not final verdicts.
