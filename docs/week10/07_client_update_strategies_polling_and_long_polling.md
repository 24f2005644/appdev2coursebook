# Topic 07: Client Update Strategies — Polling & Long Polling

---

## 1. Context: The Problem We're Solving

From Topic 6, we established that HTTP is **client-initiated and stateless** — the server cannot spontaneously push data to the browser after a page loads. The connection closes and there's no channel back.

So how do we simulate real-time updates using only standard HTTP?

The two simplest approaches — which work within HTTP's existing constraints without any special protocols — are:

1. **Fixed-Interval Polling** — client keeps asking on a timer
2. **Long Polling** — client asks, server delays its response until data is ready

Both are **pull-based** strategies that *simulate* push by cleverly managing the timing of HTTP requests.

---

## 2. Strategy 1: Fixed-Interval Polling

### How It Works

The client uses `setInterval()` (JavaScript) or equivalent to fire an HTTP request to the server **on a repeating schedule**, regardless of whether new data is available:

```
t=0s:   Client ──── GET /api/updates ──────────► Server ◄── "No new data" ── 200 { items: [] }
t=5s:   Client ──── GET /api/updates ──────────► Server ◄── "No new data" ── 200 { items: [] }
t=10s:  Client ──── GET /api/updates ──────────► Server ◄── "No new data" ── 200 { items: [] }
t=15s:  Client ──── GET /api/updates ──────────► Server ◄── "No new data" ── 200 { items: [] }
t=20s:  Client ──── GET /api/updates ──────────► Server ◄── NEW DATA ─────── 200 { items: [...] }
```

All those requests between t=0s and t=15s are **wasted** — the server had nothing new to say.

### Client-Side Implementation (JavaScript)

```javascript
// Poll every 5 seconds for new messages
const POLL_INTERVAL_MS = 5000;

async function pollForUpdates() {
    try {
        const response = await fetch('/api/updates?since=' + lastSeenTimestamp);
        const data = await response.json();

        if (data.items.length > 0) {
            displayNewItems(data.items);
            lastSeenTimestamp = data.items.at(-1).timestamp;
        }
    } catch (err) {
        console.error('Poll failed:', err);
    }
}

// Start polling loop
setInterval(pollForUpdates, POLL_INTERVAL_MS);
```

### Server-Side Implementation (Python/Flask)

```python
@app.route('/api/updates')
def get_updates():
    since = request.args.get('since', 0)
    items = db.query("SELECT * FROM updates WHERE created_at > ?", since)
    return jsonify({ "items": items })
    # Returns immediately — empty or not
```

The server simply returns whatever it has **right now** and responds immediately.

---

### Pros of Fixed-Interval Polling

| Advantage | Detail |
| :--- | :--- |
| **Simplicity** | Easiest pattern to implement — one `setInterval` + one `fetch` |
| **Stateless server** | Server just responds to each request independently; no connection state to manage |
| **Universal compatibility** | Works over plain HTTP/1.0, no special server features required |
| **Works behind firewalls/proxies** | Standard short-lived HTTP requests pass through all network middleboxes |
| **Client controls retry** | If a poll fails, the client just waits for the next interval — self-healing |

---

### Cons of Fixed-Interval Polling

#### 1. Wasted Bandwidth & CPU

The vast majority of poll responses carry **no new data**. The ratio of empty to useful responses can be enormous:

```
Scenario: New message arrives once per hour, polled every 5 seconds.
Requests per hour = 3600 / 5 = 720 requests
Useful responses   = 1
Wasted responses   = 719  (99.86% waste)
```

#### 2. Latency is Bounded by Poll Interval

If you poll every 5 seconds, your worst-case latency for receiving new data is **5 seconds** — even if the event happened 1ms after the last poll. This feels stale for real-time applications.

Reducing the interval to 1 second makes it feel more real-time but **multiplies server load by 5×**.

#### 3. The Thundering Herd / Server Overload

Consider a live sports score dashboard with **50,000 concurrent users**, all polling every 5 seconds:

```
Requests per second = 50,000 users / 5 seconds = 10,000 req/s
```

During a routine play where the score doesn't change, all 10,000 req/s return empty responses. This is enormous, unnecessary server load — and it scales linearly with users.

```
100,000 users  polling every 5s → 20,000 req/s (all empty during no-score periods)
1,000,000 users polling every 5s → 200,000 req/s (catastrophic)
```

#### 4. Not Truly Real-Time

Fixed-interval polling is an **approximation** of real-time — at best, updates arrive one poll-interval late.

---

### Summary Diagram: Fixed Polling

```
CLIENT          │    SERVER
────────────────┼────────────────────────────────────────────
GET /updates    │──► Process: no data ──► 200 { items: [] }     ← wasted
                │
[5s wait]       │
                │
GET /updates    │──► Process: no data ──► 200 { items: [] }     ← wasted
                │
[5s wait]       │
                │
[event occurs]  │    [new data now available on server]
                │
GET /updates    │──► Process: data!   ──► 200 { items: [...] }  ← useful (finally)
```

---

## 3. Strategy 2: Long Polling

### The Core Insight

Long polling is a clever refinement: instead of the server responding **immediately** (whether or not it has data), the server **holds the request open** and waits until:
- New data is actually available → respond with the data, or
- A **timeout** occurs (e.g., 30 seconds) → respond with empty/timeout signal → client immediately sends another request

The client is always "waiting" at the server — the moment data becomes available, it's delivered with **no additional delay**.

---

### How It Works — Step by Step

```
t=0s:    Client ──── GET /api/updates?wait=30 ──────────────► Server
         [Server holds connection open — no response yet]
         [Server checks DB / listens for events internally...]
         [No data at t=5s, t=10s, t=15s, t=20s...]

t=22s:   [New event arrives on server!]
         Server ────────────────────────────────────────────► Client: 200 { items: [...] }
         [Connection closes — data delivered immediately!]

t=22s:   Client ──── GET /api/updates?wait=30 ──────────────► Server
         [Client immediately re-subscribes for the next event]
         ...
```

Compare: with 5-second polling, the same event would have been received at t=25s (the next poll). Long polling delivered it at t=22s — exactly when it happened.

---

### Client-Side Implementation (JavaScript)

```javascript
// Long polling — recursive pattern
async function longPoll() {
    try {
        // Server will hold this request open for up to 30s
        const response = await fetch('/api/updates?wait=30&since=' + lastSeenTimestamp);
        const data = await response.json();

        if (data.items && data.items.length > 0) {
            displayNewItems(data.items);
            lastSeenTimestamp = data.items.at(-1).timestamp;
        }
        // No data (timeout) or got data — immediately send next request
    } catch (err) {
        // Network error — wait briefly then retry
        await new Promise(resolve => setTimeout(resolve, 1000));
    } finally {
        longPoll();  // Immediately queue next long poll (recursive)
    }
}

longPoll();  // Start the long polling loop
```

**Note the recursion**: the client is never idle. As soon as a response arrives (empty or not), it immediately fires the next long poll request. There is always one pending request at the server.

---

### Server-Side Implementation (Python/Flask)

```python
import time
from flask import Flask, request, jsonify

app = Flask(__name__)

TIMEOUT_SECONDS = 30
POLL_INTERVAL_SECONDS = 0.5

@app.route('/api/updates')
def long_poll_updates():
    since = float(request.args.get('since', 0))
    deadline = time.time() + TIMEOUT_SECONDS

    while time.time() < deadline:
        # Check if new data is available
        items = db.query("SELECT * FROM updates WHERE created_at > ?", since)

        if items:
            return jsonify({ "items": items })  # Data ready — respond!

        # No data yet — sleep briefly and check again
        time.sleep(POLL_INTERVAL_SECONDS)

    # Timeout — respond with empty to let client re-subscribe
    return jsonify({ "items": [] })
```

**Important**: This is a simplified example. In production, you'd use **event-driven mechanisms** (e.g., database `LISTEN/NOTIFY` in Postgres, Redis pub/sub, asyncio event loops, or Celery signals) rather than a busy-wait sleep loop — which wastes CPU even while "waiting".

---

### Pros of Long Polling

| Advantage | Detail |
| :--- | :--- |
| **Near real-time response** | Data is pushed the **instant** it becomes available — no waiting for the next poll interval |
| **Zero empty responses** | The server only responds when it has data (or on timeout) — no wasted bandwidth |
| **Standard HTTP compatible** | No special protocols — works over HTTP/1.1 without WebSocket upgrades |
| **Works through proxies/firewalls** | Unlike WebSockets, long polling uses standard HTTP — passes all middleboxes |
| **Widely supported** | Works in any browser; was the gold standard before SSE and WebSockets |

---

### Cons of Long Polling

#### 1. Server Threads / Connections Are Held Open

In traditional **thread-per-request** web servers (e.g., gunicorn with sync workers, older Tomcat configurations), each long-poll request holds a **worker thread** for up to 30 seconds:

```
50,000 concurrent users → 50,000 long-poll connections held open
50,000 worker threads occupied for 30s each

Typical server thread pool: 100–500 threads
→ Severe thread exhaustion with large user counts
```

**Mitigation**: Use **async / non-blocking** server architectures (e.g., `asyncio` with `aiohttp`/FastAPI, Node.js's event loop, Tornado, Nginx + async workers) where one OS thread can handle thousands of sleeping long-poll connections via I/O multiplexing.

#### 2. Timeout Handling Adds Complexity

The client must correctly handle:
- **Timeout response** → immediately re-subscribe (not treat as an error)
- **Network error** → wait and retry with backoff (avoid retry storm)
- **Data response** → process and re-subscribe

This state machine is more complex than a simple `setInterval` poll.

#### 3. Latency Under Load

When the server is heavily loaded, the internal check loop may introduce slight delays even when data is available. Under very high concurrency, the `sleep` polling inside the server adds latency.

#### 4. HTTP/1.1 Connection Limits

Browsers limit the number of concurrent HTTP connections **per domain** (typically 6 in HTTP/1.1). If a long-poll connection is always open, it consumes one of those 6 slots — potentially starving other resource requests.

**HTTP/2 multiplexing** largely resolves this since all requests share a single connection.

---

### References & Demos

- **javascript.info — Long Polling**: [https://javascript.info/long-polling](https://javascript.info/long-polling) — Excellent interactive tutorial with a live chat demo using long polling
- **Simplechat (Replit demo)**: A minimal chat app demonstrating long polling in practice

---

## 4. Direct Comparison: Fixed Polling vs. Long Polling

| Dimension | Fixed Polling | Long Polling |
| :--- | :--- | :--- |
| **Response timing** | After each fixed interval (e.g., every 5s) | Immediately when data is available |
| **Latency** | Up to 1 full poll interval | Near-zero (event-driven) |
| **Empty responses** | Majority (bandwidth waste) | Zero (only respond with data or timeout) |
| **Server connection hold** | No (short requests) | Yes (for up to timeout duration) |
| **Server thread consumption** | Low per request; high in aggregate | High per connection (with sync workers) |
| **Client complexity** | Simple `setInterval` | Recursive async function + state handling |
| **Works through HTTP proxies** | ✅ Yes | ✅ Yes |
| **Scales to 1M+ users** | ❌ Kills server | ⚠️ Needs async server |
| **Best for** | Low-frequency updates, prototypes | Near-real-time on standard HTTP |

---

## 5. When to Use Each

| Scenario | Best Strategy |
| :--- | :--- |
| Status check that updates rarely (e.g., once/hour) | ✅ Fixed Polling (simplicity wins) |
| Simple prototype or internal tool | ✅ Fixed Polling |
| Chat / notifications needing near-real-time feel, HTTP-only | ✅ Long Polling |
| Can't use WebSockets (corporate firewall/proxy restrictions) | ✅ Long Polling |
| High-frequency updates (live prices, collaborative editing) | ✅ SSE or WebSockets (Topics 8/4) |
| Server push to mobile when app is closed | ✅ Push Notifications (Topic 9) |

---

## 6. Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                  POLLING STRATEGIES AT A GLANCE                      │
├───────────────────────────┬──────────────────────────────────────────┤
│   FIXED POLLING           │   LONG POLLING                           │
├───────────────────────────┼──────────────────────────────────────────┤
│ Client asks every N secs  │ Client asks; server waits for data        │
│ Server responds instantly │ Server responds when data is ready        │
│ Many empty responses      │ Zero empty responses                      │
│ Simple setInterval        │ Recursive async + timeout handling        │
│ Latency = poll interval   │ Latency ≈ 0 (event-driven)               │
│ Stateless server          │ Holds connections (needs async server)    │
│ Works everywhere          │ Works everywhere (standard HTTP)          │
└───────────────────────────┴──────────────────────────────────────────┘
```

> **Next up:** A cleaner, more efficient solution for one-way server push → Topic 8: Server-Sent Events (SSE).
