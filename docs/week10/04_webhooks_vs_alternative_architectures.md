# Topic 04: Webhooks vs. Alternative Architectures



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 04: Webhooks vs. Alternative Architectures**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## Overview

Webhooks are not the only mechanism for real-time data delivery between systems. Understanding when to use webhooks — and when *not* to — requires comparing them against four major alternatives:

1. **WebSockets** — persistent full-duplex connections
2. **Pub/Sub Systems** — broker-mediated fan-out messaging
3. **Polling** — client-initiated periodic checks
4. **Traditional REST APIs** — on-demand data retrieval

---

## 1. Webhooks vs. WebSockets

### What Are WebSockets?
WebSockets provide a **persistent, full-duplex, bidirectional communication channel** between a client and a server over a single long-lived TCP connection.

- Protocol: `ws://` (unencrypted) or `wss://` (encrypted over TLS)
- Connection lifecycle: Initiated by the client with an HTTP Upgrade handshake, then the TCP connection remains **open indefinitely**
- Both sides can send messages **at any time**, independently of each other

### Side-by-Side Comparison

| Dimension | Webhooks | WebSockets |
| :--- | :--- | :--- |
| **Communication Direction** | One-way (Server → Server) | Full-duplex (both sides simultaneously) |
| **Who Initiates** | External server calls your endpoint | Client initiates; both communicate freely after |
| **Connection Lifecycle** | Ephemeral (one HTTP request per event) | Persistent open connection (stays alive) |
| **Protocol** | Standard HTTP/HTTPS | WebSocket protocol (`ws://` / `wss://`) |
| **Participants** | Server-to-Server (machine-to-machine) | Typically Server-to-Client (browser/app) |
| **Infrastructure** | Uses existing HTTP stack | Requires WebSocket-capable server & client |
| **State** | Stateless per event | Stateful (connection is maintained) |
| **Best For** | Asynchronous event notifications | Real-time interactive apps (chat, games, live feeds) |

### Communication Flow Diagrams

**Webhook (One-Way, Ephemeral):**
```
GitHub ──── POST /your-webhook ──────► Your App   ← single HTTP request, closes immediately
GitHub ◄──── 200 OK ─────────────────── Your App
                  (connection closed)

...[later, another event]...

GitHub ──── POST /your-webhook ──────► Your App   ← new HTTP request for each event
```

**WebSocket (Full-Duplex, Persistent):**
```
Browser ──── HTTP Upgrade: websocket ──► Server     ← handshake to establish connection
Browser ◄─── 101 Switching Protocols ── Server

──────── CONNECTION STAYS OPEN INDEFINITELY ────────

Browser ──── "Hello!" ───────────────► Server     ← client sends
Browser ◄─── "Hi back!" ──────────────── Server     ← server sends
Browser ◄─── "New message from Bob" ─── Server     ← server pushes unprompted
Browser ──── "Got it, thanks" ────────► Server     ← client responds
```

### When to Use Which:

| Use Case | Right Tool |
| :--- | :--- |
| GitHub notifying your CI/CD pipeline of a push | ✅ **Webhook** |
| Stripe notifying your app of a payment completion | ✅ **Webhook** |
| Live chat application (users messaging each other) | ✅ **WebSocket** |
| Collaborative document editing (Google Docs-style) | ✅ **WebSocket** |
| Multiplayer game state synchronisation | ✅ **WebSocket** |
| Stock ticker feed in a trading dashboard | ✅ **WebSocket** |

---

## 2. Webhooks vs. Pub/Sub Systems

### What Is Pub/Sub?
**Publish/Subscribe** is a messaging pattern mediated by a **central broker**:
- **Publishers** push messages to named **topics** on the broker (without knowing who the receivers are).
- **Subscribers** register interest in specific topics; the broker delivers messages to all matching subscribers.
- The broker provides **persistence, replay, fan-out, ordering guarantees**, and **retry semantics**.

Examples: **Google Cloud Pub/Sub**, **Apache Kafka**, **AWS SNS/SQS**, **RabbitMQ (topic exchanges)**.

### Side-by-Side Comparison

| Dimension | Webhooks | Pub/Sub |
| :--- | :--- | :--- |
| **Broker Requirement** | ❌ None — direct HTTP between parties | ✅ Requires a shared central broker |
| **Infrastructure** | Existing HTTPS stack | Dedicated broker cluster (managed or self-hosted) |
| **Fan-Out** | One endpoint per subscriber (configured manually) | Broker auto-delivers to all subscribers |
| **Delivery Guarantee** | Best-effort (HTTP timeout, manual retry) | At-least-once or exactly-once with ACKs |
| **Message Retention** | None — fire and forget | Messages stored/replayed for late subscribers |
| **Ordering** | Not guaranteed | Configurable (partition-level ordering in Kafka) |
| **Scale** | Hundreds to thousands of subscribers is complex | Millions of messages/sec natively |
| **Best For** | Simple cross-org event notifications | High-volume internal event streaming & fan-out |

### Communication Flow Diagrams

**Webhook (Direct Push, No Broker):**
```
GitHub ──── POST https://app1.com/hook ──────► App 1
GitHub ──── POST https://app2.com/hook ──────► App 2
GitHub ──── POST https://app3.com/hook ──────► App 3
            (GitHub calls each registered URL individually)
```

**Pub/Sub (Broker-Mediated Fan-Out):**
```
Publisher ──── publish("push-event", payload) ──► [ Broker / Topic ]
                                                         │
                                          ┌──────────────┼──────────────┐
                                          ▼              ▼              ▼
                                      Subscriber 1  Subscriber 2  Subscriber 3
                                      (CI/CD)       (Slack Bot)   (Analytics)
```

### When to Use Which:

| Use Case | Right Tool |
| :--- | :--- |
| GitHub notifying external apps of events | ✅ **Webhook** |
| Internal microservices reacting to order events | ✅ **Pub/Sub** |
| 1M+ events/sec (IoT sensor data processing) | ✅ **Pub/Sub (Kafka)** |
| Replay old events for a new downstream service | ✅ **Pub/Sub** |
| Simple Stripe → your app payment notification | ✅ **Webhook** |

---

## 3. Webhooks vs. Polling

### What Is Polling?
**Polling** is the client-initiated pattern of periodically sending requests to a server to check whether new data or a status change is available.

- **Fixed-interval polling**: Client sends a request every N seconds regardless of whether new data exists.
- **Long polling**: Client sends a request; server holds it open until data is available or a timeout occurs (covered in Topic 7).

### Side-by-Side Comparison

| Dimension | Webhooks (Push) | Polling (Pull) |
| :--- | :--- | :--- |
| **Initiation** | Server pushes to client when event occurs | Client pulls from server on a schedule |
| **Latency** | ✅ Real-time (event-driven, immediate) | ❌ Delayed by poll interval |
| **Wasted Requests** | ✅ Zero (only called on actual events) | ❌ Many empty responses between events |
| **Server Load** | ✅ Low (proportional to actual events) | ❌ High (proportional to clients × poll frequency) |
| **Client Complexity** | ❌ Requires a public HTTPS endpoint | ✅ Simple — just a timed HTTP request |
| **Reliability** | ❌ Requires endpoint uptime | ✅ Client controls retry |
| **Scalability** | ✅ Better at scale | ❌ Poor — N clients × M polls/sec overwhelms servers |

### The Scaling Problem With Polling

Consider a scenario where **10,000 clients** each poll a status endpoint every **5 seconds**:

```
Requests per second = 10,000 clients × (1 request / 5 seconds) = 2,000 req/s
Empty responses (no new data 99% of the time) ≈ 1,980 wasted req/s
```

With webhooks, the server only makes requests when an actual event occurs — which might be **10 events per second** across all 10,000 clients. The efficiency ratio is enormous.

### Visual Contrast:

**Polling:**
```
Client ──── GET /status ──► Server   [empty: "pending"]   (t=0s)
Client ──── GET /status ──► Server   [empty: "pending"]   (t=5s)
Client ──── GET /status ──► Server   [empty: "pending"]   (t=10s)
Client ──── GET /status ──► Server   [empty: "pending"]   (t=15s)
...17 more wasted requests...
Client ──── GET /status ──► Server   ["completed!"]       (t=100s)
```

**Webhook:**
```
...Silence (zero requests)...
Server ──── POST /your-callback ──► Client   ["completed!"]    (t=100s, instantly)
```

### When to Use Which:

| Use Case | Right Tool |
| :--- | :--- |
| Your app can expose a public HTTPS endpoint | ✅ **Webhook** |
| You're querying a service that doesn't support webhooks | ✅ **Polling** |
| Quick prototype / simple script checking a status once | ✅ **Polling** |
| Production system with many clients needing real-time updates | ✅ **Webhook** |
| Client is behind a NAT/firewall with no public endpoint | ✅ **Polling** |

---

## 4. Webhooks vs. Traditional REST APIs

### What Is a Traditional REST API?
A REST API is a request-response interface where a **client actively queries** a server to **retrieve data or trigger an action**. The client controls *when* to ask and *what* to ask for.

### Side-by-Side Comparison

| Dimension | Webhooks | Traditional REST API |
| :--- | :--- | :--- |
| **Who Initiates** | The data provider (server) pushes to you | You (the client) pull from the provider |
| **Primary Goal** | Deliver event notifications in real-time | Retrieve or manipulate resources on demand |
| **Timing** | Triggered by events asynchronously | Triggered by explicit client requests |
| **Data Freshness** | Always current (event at the moment it happens) | As fresh as your last request |
| **Response Body** | Ignored by sender | Core value delivered to the client |
| **Use Case** | Asynchronous event-driven integrations | Synchronous data queries and CRUD operations |

### Complementary, Not Competing:
Webhooks and REST APIs are almost always used **together**:

1. **Webhook** tells you that *something happened* (e.g., "payment completed").
2. **REST API call** lets you *fetch full details* about what happened (e.g., `GET /payments/{id}`).

```
Stripe ──── POST /your-webhook ──────► Your App
            Body: { "event": "payment_intent.succeeded", "id": "pi_abc123" }

Your App ──── GET /v1/payment_intents/pi_abc123 ──────► Stripe REST API
Your App ◄─── { "amount": 5000, "currency": "usd", "customer": {...} } ── Stripe
```

---

## 5. Master Comparison Table

| Criterion | Webhooks | WebSockets | Pub/Sub | Polling | REST API |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Real-time** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Bidirectional** | ❌ | ✅ | Partial | ❌ | ❌ |
| **Server-to-Server** | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Persistent Connection** | ❌ | ✅ | Varies | ❌ | ❌ |
| **No Broker Needed** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Delivery Guarantees** | ❌ | ❌ | ✅ | N/A | N/A |
| **Message Replay** | ❌ | ❌ | ✅ | N/A | N/A |
| **Client Needs Public URL** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Complexity** | Low | Medium | High | Low | Low |
| **Best Fit** | Cross-org event push | Real-time interactive | High-volume internal | Simple scripts | On-demand data |

---

## 6. Summary

- **Use webhooks** when you need real-time, server-initiated, cross-organization event notifications over standard HTTP — and when you can expose a public endpoint.
- **Use WebSockets** when you need persistent, bidirectional, low-latency communication (e.g., chat, live collaboration, gaming).
- **Use Pub/Sub** when you need high-volume, guaranteed, fan-out message delivery within internal infrastructure.
- **Use polling** as a last resort — when the other party doesn't support webhooks, or you're behind a firewall without a public endpoint.
- **REST APIs** complement webhooks — webhooks alert, REST APIs retrieve the full details.

> **Next up:** How do you actually set up, debug, and secure a webhook integration? → Topic 5: Webhook Implementation, Debugging & Security.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

