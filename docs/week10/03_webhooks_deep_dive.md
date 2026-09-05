# Topic 03: Webhooks Deep Dive

---

## 1. What Is a Webhook? — Definition & Origin

### The Official Definition
> *"A webhook is a way for an app to provide other apps with real-time information."*

More precisely, a webhook is an **HTTP callback** — a user-defined HTTP endpoint that an external service calls ("hooks into") when a specific event occurs.

### Alternative Names
The concept is known by several synonymous names across the industry:

| Term | Why It's Used |
| :--- | :--- |
| **Webhook** | Most common; "web" (HTTP-based) + "hook" (triggered by an event) |
| **Web Callback** | Emphasises that the external service "calls back" your URL |
| **HTTP Push API** | Contrasts with pull-based APIs; data is *pushed* to your endpoint |
| **Reverse API** | Your server exposes an endpoint for another service to call — the *opposite* of typical client-server |

### Historical Background
- The term "webhook" was coined by **Jeff Lindsay** in 2007 in a blog post titled *"Web Hooks to revolutionize the web"*.
- The core idea: instead of apps polling each other, let HTTP itself carry event notifications — using infrastructure that already exists everywhere.

---

## 2. The "Reverse API" Mental Model

Understanding webhooks is easiest by contrasting them with a traditional REST API call:

### Traditional REST API (Pull / Client-Initiated):
```
Your App ──── GET /repos/vinay/project/commits ───────────────► GitHub
Your App ◄─── [ { "sha": "abc", "message": "fix bug" }, ... ] ─ GitHub
```
- *You* initiate the request.
- *You* pull data *from* GitHub *on demand*.

### Webhook (Push / Server-Initiated — "Reverse API"):
```
GitHub ──────── POST https://yourapp.com/webhook/github ──────► Your App
                Body: { "event": "push", "commits": [...] }
GitHub ◄──────── 200 OK ──────────────────────────────────────── Your App
```
- *GitHub* initiates the request.
- *GitHub* pushes event data *to you* the moment it happens.
- You are the **server**; GitHub is acting as the **client**.

This inversion of roles is why it's called a **Reverse API**.

---

## 3. How Webhooks Work — The Execution Model

### Step-by-Step Lifecycle

```
Step 1: Register
  Your App ──── "Notify me at https://yourapp.com/hooks/github on push events" ──► GitHub (via GitHub UI/API settings)

Step 2: Event Occurs
  Developer ──── git push ──────────────────────────────────────────────────────► GitHub

Step 3: Webhook Triggered
  GitHub ──────── POST https://yourapp.com/hooks/github ───────────────────────► Your App
                  Headers: { X-GitHub-Event: "push", X-Hub-Signature-256: "sha256=..." }
                  Body:    { "ref": "refs/heads/main", "commits": [...], "pusher": {...} }

Step 4: Immediate Acknowledge
  Your App ◄───── 200 OK ──────────────────────────────────────────────────────── Your App
                  (Return immediately — do NOT do heavy processing before responding)

Step 5: Background Processing
  Your App ──── (enqueue task to internal queue: log commit, trigger CI/CD, post to Slack) ──► Internal Worker
```

### Critical Execution Rule: Respond First, Process Later

This is one of the most important practical details about webhooks:

> **You MUST return an HTTP response immediately — before doing any significant processing.**

**Why?**
- The calling service (e.g., GitHub, Twilio) typically has a **response timeout** of a few seconds (commonly 5–30 seconds).
- If your endpoint takes too long to respond, the sender marks the delivery as **failed** and may retry — causing **duplicate processing**.
- Heavy work (database writes, API calls, notifications) must be handed off to a **background task/worker queue** before returning `200 OK`.

```python
# Example: Flask webhook endpoint — correct pattern
@app.route('/webhook/github', methods=['POST'])
def handle_github_push():
    payload = request.get_json()

    # 1. Immediately enqueue for background processing
    task_queue.enqueue(process_github_push, payload)

    # 2. Return 200 immediately — don't block!
    return '', 200

def process_github_push(payload):
    # This runs asynchronously in a worker
    commits = payload['commits']
    notify_slack(commits)
    trigger_ci_pipeline(commits)
    log_to_database(commits)
```

**Contrast with traditional message queues**: A message broker provides built-in persistence/retry. With webhooks, if you fail to respond in time and the sender doesn't retry, **the message is lost** — making the "respond immediately" rule even more critical.

---

## 4. Webhook Payload — Message Contents

### Key Characteristics:

1. **Entirely Application-Defined**: There is no universal webhook payload standard. Each service (GitHub, Stripe, Twilio, etc.) defines its own JSON or form-encoded schema.

2. **Kept Minimal — Notification, Not Data Transfer**:
   - The payload carries *identifiers and event metadata*, not full data dumps.
   - If you need full details (e.g., full user profile), you make a **follow-up REST API call** using the ID from the payload.

3. **Delivered in the HTTP Request Body**:
   - `Content-Type: application/json` for modern services (most common).
   - `Content-Type: application/x-www-form-urlencoded` for older/simpler services.

### Anatomy of a Real Webhook Payload (GitHub Push Event):
```json
{
  "ref": "refs/heads/main",
  "before": "a1b2c3d4",
  "after":  "e5f6g7h8",
  "repository": {
    "id": 123456,
    "name": "my-project",
    "full_name": "vinay/my-project"
  },
  "pusher": {
    "name": "vinay",
    "email": "vinay@example.com"
  },
  "commits": [
    {
      "id": "e5f6g7h8",
      "message": "Fix login bug",
      "timestamp": "2026-09-04T12:00:00Z",
      "author": { "name": "Vinay", "email": "vinay@example.com" }
    }
  ]
}
```

Notice: this tells your app *what happened* and *who did it* — but it doesn't include every piece of related data. You'd call the GitHub REST API to fetch full diffs, PR details, etc.

---

## 5. Webhook Responses — What Your Endpoint Should Return

Since webhooks are **machine-to-machine** (server calling server), the response semantics are different from browser-facing APIs:

### Response Status Codes:

| Code | Meaning | Sender's Action |
| :--- | :--- | :--- |
| `200 OK` | Successfully received and acknowledged | No retry needed |
| `201 Created` | Accepted and a resource was created | No retry needed |
| `204 No Content` | Received, no response body | No retry needed |
| `4xx` (e.g., 400, 401, 403) | Your endpoint rejected the request | Sender may log error; usually **no retry** (client error) |
| `5xx` (e.g., 500, 503) | Your server failed to process it | Sender will typically **retry** with exponential backoff |
| **Timeout (no response)** | Your server took too long | Sender treats as failure, **will retry** |

### Response Body:
- **Completely ignored by the sender** in nearly all webhook implementations.
- Your response body is **discarded** — the sender only cares about the status code.
- Keep the body empty (`''`) or minimal (`{"status": "ok"}`).

This contrasts sharply with REST APIs where the response body is the primary value delivered.

---

## 6. Webhooks Using Existing Web Infrastructure

A key advantage of webhooks over dedicated message queue systems:

### No New Infrastructure Required:
- Webhooks run over **standard HTTPS** — the same infrastructure your web app already uses.
- Your webhook receiver is just a **regular HTTP route** in your existing web framework (Flask, Django, Express, FastAPI, etc.).
- **No special broker software** to install, configure, or maintain.
- **Firewall-friendly**: Works over port 443 (HTTPS), which is almost always open.

### This Means:
- Any service with a public HTTPS endpoint can receive webhooks.
- Any service that can make HTTP POST requests can send webhooks.
- The "integration" is just sharing a URL — as simple as it gets.

---

## 7. Summary: Webhooks at a Glance

```
┌─────────────────────────────────────────────────────────────────────┐
│                        WEBHOOK SUMMARY                              │
├─────────────────┬───────────────────────────────────────────────────┤
│ Definition      │ HTTP callback triggered by events in an external  │
│                 │ service; data is pushed to your endpoint          │
├─────────────────┼───────────────────────────────────────────────────┤
│ Also Called     │ Web Callback, HTTP Push API, Reverse API          │
├─────────────────┼───────────────────────────────────────────────────┤
│ Transport       │ Standard HTTP/HTTPS (usually POST)                │
├─────────────────┼───────────────────────────────────────────────────┤
│ Payload         │ Minimal JSON/form-encoded event data in body      │
├─────────────────┼───────────────────────────────────────────────────┤
│ Response        │ Status code only (200/204); body is ignored       │
├─────────────────┼───────────────────────────────────────────────────┤
│ Critical Rule   │ Respond immediately; offload processing to        │
│                 │ background workers                                │
├─────────────────┼───────────────────────────────────────────────────┤
│ Infrastructure  │ No broker needed — uses existing web stack        │
├─────────────────┼───────────────────────────────────────────────────┤
│ Direction       │ Server → Server (external service calls your app) │
└─────────────────┴───────────────────────────────────────────────────┘
```

> **Next up:** How do webhooks compare to WebSockets, Pub/Sub, Polling, and REST APIs? → Topic 4: Webhooks vs. Alternative Architectures.
