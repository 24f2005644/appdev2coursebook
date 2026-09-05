# Topic 02: Lightweight API Calls & Messaging Patterns

---

## 1. The Problem: Cross-Organization Async Communication

From Topic 1, we established that traditional message queues work well **within** a private infrastructure. But what happens when:
- Your app needs **GitHub** to notify you whenever someone pushes a commit?
- You want **Twilio** to tell you when a bulk SMS campaign finishes sending?
- **Stripe** needs to inform your app when a payment has been processed?

In all these cases, there is no shared message broker. Instead, both parties are on the **public internet** — and communication must happen over the **universal protocol**: **HTTP**.

> The solution: **Lightweight API Calls** — one party exposes a simple HTTP endpoint, and the other party calls it to "push" a notification.

---

## 2. What Is a Lightweight API Call (for Messaging)?

In the traditional mental model of APIs (e.g., REST), a client **pulls** data from a server:

```
Client ───── GET /users/123 ────────────► Server
Client ◄──── { "id": 123, "name": "..." } ── Server
```

A **lightweight API call for messaging** flips this pattern. The purpose is not to *fetch* data but to *deliver a notification or event* from one server to another:

```
Service A ──── POST /your-webhook-endpoint ────► Service B
Service A ◄─── 200 OK (acknowledge receipt) ──── Service B
```

### Key Properties

| Property | Description |
| :--- | :--- |
| **Purpose** | PUSH a message / event to another service — not data retrieval |
| **HTTP Method** | Usually `POST` (occasionally `GET` for simple pings) |
| **Payload** | Minimal or nearly empty — just enough to identify the event |
| **Response** | A simple status code acknowledging receipt (`200 OK`) |
| **Direction** | Server → Server (machine-to-machine, no human in the loop) |

### Why `POST` and not `GET`?
- `GET` is semantically designed for *reading/requesting* data; it should be **idempotent** and **side-effect-free**.
- `POST` is designed for *sending data* that will cause an action or state change on the receiver.
- Webhook payloads are sent in the **request body**, which `GET` requests technically should not have.
- Convention: use `POST` to push event data; use `GET` only for simple "ping" liveness checks.

---

## 3. Core Motivation: Event-Driven Notification Without Polling

The fundamental reason for lightweight API call messaging is **eliminating polling**.

### The Problem With Polling

Imagine you use Twilio to send 50,000 SMS messages as part of a campaign. How does your app know when it's finished?

**Option A — Polling (Bad):**
```
Your App ── GET /campaign/status ─────► Twilio  (every 5 seconds)
Your App ◄── { "status": "in_progress" } ─ Twilio  (2000 wasted requests later)
Your App ◄── { "status": "completed" }  ─ Twilio  (finally!)
```

Problems:
- Wastes bandwidth, CPU, and API rate limits.
- Introduces latency (you only find out at the next poll interval).
- Doesn't scale: 10,000 apps polling Twilio simultaneously = service meltdown.

**Option B — Lightweight API Call / Webhook (Good):**
```
Your App ── "Call me back at https://yourapp.com/twilio-done" ──► Twilio
...Twilio processes campaign in background...
Twilio ──── POST https://yourapp.com/twilio-done ─────────────► Your App
Your App ◄── 200 OK ────────────────────────────────────────── Your App
```

Result:
- Zero wasted requests.
- Notification arrives the **instant** the event completes.
- Twilio only makes **one** HTTP call per completed campaign.

---

## 4. Real-World Examples & Scenarios

### Example A: GitHub Commits → Google Chat Notification

**Workflow:**
1. A developer pushes code to a GitHub repository.
2. GitHub detects the `push` event.
3. GitHub makes a `POST` request to a pre-configured webhook URL (e.g., a Google Chat Incoming Webhook URL).
4. Google Chat receives the payload, formats it, and posts a message to the team's chat room.

```
Developer ──── git push ─────────────────────────────────────────► GitHub
GitHub ──────── POST https://chat.googleapis.com/v1/spaces/.../messages ──► Google Chat
Payload: { "text": "Vinay pushed 3 commits to main: fix login bug, add tests..." }
```

**Why this is a "lightweight" API call:**
- GitHub is calling Google Chat's generic "post a message" endpoint.
- The payload is a small JSON object — no database dumps or large transfers.
- Google Chat just responds `200 OK`; no complex reply data needed.

---

### Example B: Twilio Bulk Messaging → Async Completion Callback

**Without callback (polling anti-pattern):**
```
Your App ──── POST /messages/bulk ────────────────────────────► Twilio (start campaign)
Your App ──── GET /campaigns/abc123 ──────────────────────────► Twilio (check every 5s)
Your App ──── GET /campaigns/abc123 ──────────────────────────► Twilio (still pending...)
... repeated many times ...
Your App ◄─── { "status": "completed" } ─────────────────────── Twilio (finally)
```

**With lightweight API callback (webhook):**
```
Your App ──── POST /messages/bulk ─────────────────────────────► Twilio
              Body: { ..., "statusCallback": "https://myapp.com/twilio/callback" }
Twilio ──────── (processes 50,000 SMS in background) ...
Twilio ──────── POST https://myapp.com/twilio/callback ─────────► Your App
                Body: { "campaignId": "abc123", "status": "completed", "sent": 49982 }
Your App ◄───── 200 OK ──────────────────────────────────────── Your App
```

The key detail: **your app registers its own callback URL** as part of the initial request to Twilio. This is the essence of the webhook pattern.

---

## 5. The Anatomy of a Lightweight Messaging API Call

```
POST /webhook-receiver HTTP/1.1
Host: yourapp.com
Content-Type: application/json
X-Source-Service: github          ← optional: identifies sender
X-Signature-256: sha256=abc123    ← optional: for security verification

{
  "event": "push",
  "ref": "refs/heads/main",
  "pusher": { "name": "vinay" },
  "commits": [ ... ]
}
```

### Components Breakdown:

| Component | Role |
| :--- | :--- |
| **Endpoint URL** | A simple, publicly accessible HTTPS endpoint on the receiver's server |
| **HTTP Method** | `POST` (payload in body) or `GET` (for simple ping events) |
| **Headers** | Content-Type, optional security tokens/signatures |
| **Payload (Body)** | Event data in JSON or form-encoded format — kept minimal |
| **Response** | `200 OK` or `204 No Content` = success; `4xx`/`5xx` = error/retry |

---

## 6. Lightweight vs. Heavy Messaging: At a Glance

| Dimension | Lightweight API Call / Webhook | Traditional Message Queue |
| :--- | :--- | :--- |
| **Infrastructure** | Standard HTTP — no shared broker needed | Requires shared broker (RabbitMQ, Kafka) |
| **Delivery Model** | Immediate synchronous HTTP push | Asynchronous queue-based |
| **Payload Size** | Small notification / event data | Can support large payloads |
| **Delivery Guarantee** | Best-effort (sender retries on non-200) | At-least-once / exactly-once semantics |
| **Ordering** | Not guaranteed | Can be guaranteed (FIFO queues) |
| **Who Can Use** | Any service that can make/receive HTTP | Must share infrastructure / trust model |
| **Best For** | Cross-org event notifications | Internal microservice workflows |

---

## 7. Summary

- Lightweight API calls solve the fundamental problem of **pushing event notifications** across organizational boundaries over the public internet.
- The key insight is: **you expose an HTTP endpoint** and register it with the third-party service. The third-party **calls your endpoint** when the event occurs — this is the "reverse API" / webhook concept.
- The payload is deliberately **minimal** — just enough to identify the event and relevant IDs. Your app can then make a subsequent API call to fetch full details if needed.
- This pattern eliminates **polling** entirely, resulting in real-time responsiveness with zero wasted requests.

> This sets the foundation for understanding **Webhooks** (Topic 3), which is the formalized, industry-standard implementation of this lightweight messaging pattern.
