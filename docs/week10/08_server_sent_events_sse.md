# Topic 08: Server-Sent Events (SSE)



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 08: Server-Sent Events (SSE)**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 1. Motivation: Why SSE Over Long Polling?

From Topic 7, we saw that long polling *simulates* push by cleverly holding HTTP connections open. But it has friction:

- Every event delivery **closes the connection** — the client must immediately re-open it.
- The server must manage a **half-open TCP connection** that gets torn down and re-established constantly.
- Complex client-side retry and timeout state management.
- No standard wire format — every implementation rolls its own conventions.

**Server-Sent Events (SSE)** solves all of these by making **streaming push a first-class, standardised HTTP feature** — one single persistent connection over which the server can stream as many events as it wants, whenever it wants.

---

## 2. What Is SSE? — Definition & Standard

**Server-Sent Events** is a W3C standard (part of the HTML5 specification) that defines:
1. A **wire format** for server-to-client event streams over HTTP
2. A browser-native **`EventSource` API** that handles the connection, parsing, and automatic reconnection

> *"SSE is a server push technology enabling a client to receive automatic updates from a server via an HTTP connection."*

Key properties at a glance:

| Property | Value |
| :--- | :--- |
| **Direction** | Unidirectional — Server → Client only |
| **Protocol** | Standard HTTP/HTTPS (no upgrade required) |
| **Content-Type** | `text/event-stream` |
| **Connection** | One long-lived persistent connection per client |
| **Browser API** | `EventSource` (built into all modern browsers) |
| **Auto-reconnect** | ✅ Built-in — browser reconnects automatically on disconnect |
| **W3C Standard** | ✅ Yes — part of HTML5 Living Standard |

---

## 3. The SSE Wire Format

SSE uses a simple, human-readable **text-based protocol** over the HTTP response body. The server keeps the response body open and writes events as they occur.

### Format Rules:

- Events are separated by **two newlines** (`\n\n`)
- Each event can have multiple fields, one per line: `field: value\n`
- Standard fields:

| Field | Purpose | Example |
| :--- | :--- | :--- |
| `data` | The event payload (required) | `data: {"score": 42}` |
| `event` | Named event type (optional) | `event: score-update` |
| `id` | Event ID for reconnect tracking | `id: 1042` |
| `retry` | Reconnect delay in ms (optional) | `retry: 3000` |
| `:` (comment) | Keep-alive heartbeat / comment | `: ping` |

### Example Event Stream:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

: ping\n
\n
id: 1\n
event: message\n
data: {"user": "Vinay", "text": "Hello!"}\n
\n
id: 2\n
event: score-update\n
data: {"team": "India", "score": 150}\n
\n
id: 3\n
data: Simple message with no event type\n
\n
```

Each double-newline `\n\n` signals the end of one event — the browser's `EventSource` parses and fires the appropriate JavaScript event handler.

---

## 4. How SSE Works — End-to-End Flow

### Connection Lifecycle

```
Step 1: Client Opens SSE Connection
  Browser ──── GET /api/events ──────────────────────────────► Server
               Accept: text/event-stream

Step 2: Server Sends Headers, Keeps Connection Open
  Browser ◄─── HTTP 200 OK ──────────────────────────────────── Server
               Content-Type: text/event-stream
               Cache-Control: no-cache
               [Connection stays open — response body streams continuously]

Step 3: Server Pushes Events as They Occur
  Browser ◄─── "data: {score: 1}\n\n" ──────────────────────── Server  (t=5s)
  Browser ◄─── "data: {score: 2}\n\n" ──────────────────────── Server  (t=12s)
  Browser ◄─── ": ping\n\n" ─────────────────────────────────── Server  (t=30s, keepalive)
  Browser ◄─── "data: {score: 3}\n\n" ──────────────────────── Server  (t=47s)

Step 4: Auto-Reconnect on Disconnect
  [Network hiccup — connection drops]
  Browser ──── GET /api/events ──────────────────────────────► Server
               Last-Event-ID: 2     ← Browser sends last received ID
  [Server resumes from event ID 3 — no missed events]
```

**Compare to long polling**: In long polling, after every event the connection closes and must be reopened. With SSE, the **connection stays open indefinitely** and events stream continuously — much more efficient.

---

## 5. Browser-Side: The `EventSource` API

The browser provides a native `EventSource` object that abstracts all the connection management:

### Basic Usage

```javascript
// Open SSE connection — browser handles everything automatically
const eventSource = new EventSource('/api/events');

// Listen for default 'message' events (events without an 'event:' field)
eventSource.onmessage = function(event) {
    const data = JSON.parse(event.data);
    console.log('New message:', data);
    updateUI(data);
};

// Listen for named event types (matching 'event: score-update' in the stream)
eventSource.addEventListener('score-update', function(event) {
    const score = JSON.parse(event.data);
    document.getElementById('score').textContent = score.team + ': ' + score.score;
});

// Handle connection errors / reconnection
eventSource.onerror = function(err) {
    console.error('SSE error:', err);
    // EventSource automatically reconnects — no manual retry needed!
};

// Close the connection when done (e.g., user navigates away)
// eventSource.close();
```

### EventSource Properties

| Property/Method | Description |
| :--- | :--- |
| `new EventSource(url)` | Opens SSE connection to the given URL |
| `new EventSource(url, { withCredentials: true })` | Include cookies / auth credentials |
| `.onmessage` | Handler for unnamed events |
| `.addEventListener(type, fn)` | Handler for named event types |
| `.onerror` | Handler for errors |
| `.readyState` | `0` (Connecting), `1` (Open), `2` (Closed) |
| `.close()` | Permanently close the connection |

---

## 6. Server-Side: Implementing an SSE Endpoint

### Python / Flask

```python
import time
import json
from flask import Flask, Response, stream_with_context

app = Flask(__name__)

def generate_events():
    """Generator that yields SSE-formatted events."""
    event_id = 0
    while True:
        # Check for new data (e.g., from DB or event queue)
        new_data = get_new_data_from_queue()

        if new_data:
            event_id += 1
            payload = json.dumps(new_data)
            # SSE format: id, event type, data, blank line
            yield f"id: {event_id}\n"
            yield f"event: update\n"
            yield f"data: {payload}\n\n"
        else:
            # Keep-alive comment — prevents proxy/browser timeout
            yield ": ping\n\n"

        time.sleep(1)

@app.route('/api/events')
def sse_endpoint():
    return Response(
        stream_with_context(generate_events()),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'X-Accel-Buffering': 'no'   # Disable Nginx buffering for SSE
        }
    )
```

### Node.js / Express

```javascript
app.get('/api/events', (req, res) => {
    // Set SSE headers
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.flushHeaders();  // Send headers immediately

    let eventId = 0;

    // Send an event every time new data arrives
    const interval = setInterval(() => {
        const data = getLatestData();
        if (data) {
            eventId++;
            res.write(`id: ${eventId}\n`);
            res.write(`event: update\n`);
            res.write(`data: ${JSON.stringify(data)}\n\n`);
        } else {
            res.write(': ping\n\n');  // Keepalive
        }
    }, 1000);

    // Clean up when client disconnects
    req.on('close', () => {
        clearInterval(interval);
        res.end();
    });
});
```

---

## 7. The Role of Web Workers & Service Workers

The skeleton mentions **Web Workers** and **Service Workers** in the context of SSE. Here's how they relate:

### Web Workers — Background Thread for SSE Processing

A **Web Worker** is a JavaScript thread running in the **background** — separate from the main UI thread. Normally, JavaScript is single-threaded; heavy work on the main thread blocks UI rendering.

**Using a Web Worker with SSE:**

```javascript
// main.js (runs on main thread)
const worker = new Worker('sse-worker.js');

worker.onmessage = function(event) {
    // Receive processed SSE data from the worker
    updateUI(event.data);
};
```

```javascript
// sse-worker.js (runs in background thread)
const eventSource = new EventSource('/api/events');

eventSource.onmessage = function(event) {
    const data = JSON.parse(event.data);
    // Heavy processing here (doesn't block UI)
    const processed = heavyProcessing(data);
    // Send result back to main thread
    postMessage(processed);
};
```

**Why this matters for SSE:**
- SSE events may arrive continuously at high frequency (e.g., real-time market data).
- Processing and parsing each event on the main thread would cause **UI jank** (dropped frames, laggy interactions).
- Offloading to a Web Worker means the UI stays **smooth and responsive** even while processing high-frequency event streams.

### Service Workers — Background Push Even When Page Is Closed

A **Service Worker** is a more powerful script that runs as a **proxy between the browser and the network**, independent of any specific page. It can run even when the user has closed the tab.

**Relation to SSE:**
- Service Workers are primarily associated with **Push Notifications** (Topic 9) rather than SSE directly.
- However, Service Workers can intercept `EventSource` connections and cache/relay events.
- More practically: SSE works while the page is open; for background push when the page is closed, you need Service Workers + the Push API.

| | Web Worker | Service Worker |
| :--- | :--- | :--- |
| **Lifecycle** | Lives as long as the page is open | Can live after page is closed |
| **Scope** | Per-page background thread | Browser-level background script |
| **Use with SSE** | Process high-frequency events off UI thread | Can intercept/proxy network requests |
| **Use with Push** | Not directly used | ✅ Core mechanism for background push |
| **Network access** | ✅ Can open EventSource | ✅ Intercepts all fetch/network calls |

---

## 8. SSE vs. Long Polling vs. WebSockets

| Dimension | Long Polling | SSE | WebSockets |
| :--- | :--- | :--- | :--- |
| **Direction** | Server → Client (simulated) | Server → Client (native) | Full-duplex (both ways) |
| **Protocol** | Plain HTTP | Plain HTTP (`text/event-stream`) | WebSocket protocol (`ws://`) |
| **Connection** | Re-established per event | One persistent connection | One persistent connection |
| **Browser API** | Manual `fetch()` loop | Native `EventSource` | Native `WebSocket` |
| **Auto-reconnect** | ❌ Manual (client code) | ✅ Built-in | ❌ Manual |
| **Event IDs / Resume** | ❌ Manual tracking | ✅ Built-in (`Last-Event-ID`) | ❌ Manual |
| **Firewall/Proxy** | ✅ Standard HTTP | ✅ Standard HTTP | ⚠️ Some proxies block `ws://` |
| **HTTP/2 support** | ✅ | ✅ (multiplexed efficiently) | N/A (separate protocol) |
| **Server complexity** | Medium | Low-Medium | Medium-High |
| **Best for** | Legacy compatibility | One-way real-time streams | Bidirectional real-time apps |

### SSE's Key Advantages Over Long Polling:
1. **No reconnection overhead** — one persistent connection for the lifetime of the session.
2. **Standardized wire format** — no custom protocol to design or maintain.
3. **Built-in reconnect + `Last-Event-ID`** — the browser automatically reconnects and the server knows exactly where to resume, preventing missed events.
4. **Simpler server code** — stream from a generator/async loop; no need to manage connection teardown/re-establishment logic.

### When SSE Loses to WebSockets:
- If the client also needs to **send data** to the server frequently (e.g., chat messages, game inputs, collaborative edits), SSE is one-way only. WebSockets provide full-duplex.

---

## 9. Trade-offs & Limitations of SSE

| Trade-off | Detail |
| :--- | :--- |
| **Unidirectional only** | Client cannot send data over the SSE connection — must use separate `fetch`/`POST` for client → server |
| **Long-lived server connections** | Like long polling, each connected client holds an open connection — requires async server architecture (FastAPI, Node.js, asyncio) to scale |
| **Proxy / corporate firewall buffering** | Some HTTP proxies buffer responses, breaking the streaming nature. Requires `X-Accel-Buffering: no` for Nginx, or using `wss://` WebSockets to avoid |
| **Browser connection limit (HTTP/1.1)** | Max 6 concurrent connections per domain in HTTP/1.1 — SSE consumes one permanently. HTTP/2 multiplexing resolves this |
| **No binary data** | SSE is text-only (`text/event-stream`). Binary data must be Base64-encoded |
| **IE/Edge Legacy** | Older IE had no native `EventSource` support (polyfills available). All modern browsers support it |

---

## 10. SSE in the Wild — Real-World Uses

| Application | How SSE Is Used |
| :--- | :--- |
| **GitHub Actions** | Live build log streaming in the browser — each log line pushed as an SSE event |
| **ChatGPT / LLM streaming** | AI-generated text streamed token-by-token via SSE — that "typing" effect is SSE |
| **Live sports dashboards** | Score and stat updates pushed to all viewers simultaneously |
| **Stock / crypto tickers** | Price updates streamed continuously without polling |
| **Progress bars** | Long-running server job (e.g., export, report generation) streams progress % via SSE |
| **Notifications in SPAs** | In-app notification badges updated in real-time without full-page reload |

---

## 11. Summary

```
┌───────────────────────────────────────────────────────────────────────┐
│                    SERVER-SENT EVENTS AT A GLANCE                     │
├───────────────────────────────────────────────────────────────────────┤
│  What it is:   W3C-standardised server-to-client event streaming      │
│                over a single persistent HTTP connection               │
│                                                                       │
│  Wire format:  text/event-stream — "data: ...\n\n" per event          │
│                                                                       │
│  Browser API:  EventSource — one line to connect, auto-reconnects     │
│                                                                       │
│  Direction:    One-way only (Server → Client)                         │
│                                                                       │
│  Advantages over long polling:                                        │
│    ✅ No per-event reconnect overhead                                 │
│    ✅ Built-in auto-reconnect + Last-Event-ID (resume after drop)     │
│    ✅ Standardised format — no custom protocol                        │
│    ✅ Works through firewalls (standard HTTP)                         │
│                                                                       │
│  Limitations:                                                         │
│    ❌ One-way only — can't send data from client to server over SSE   │
│    ❌ Needs async server to handle many concurrent streams            │
│    ❌ Text-only (binary requires Base64 encoding)                     │
│                                                                       │
│  Web Workers: Offload SSE event processing off the UI thread          │
│  Service Workers: For push when the page is closed → see Topic 9     │
└───────────────────────────────────────────────────────────────────────┘
```

> **Next up (and final topic):** What about pushing to browsers/devices when the tab is closed? → Topic 9: Push Notifications & Modern Push Protocols.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

