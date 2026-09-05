# Topic 2: Architecture for Heavy Compute & Async Task Frameworks

---

## 2.1 Core Architectural Separation

### The Fundamental Problem

As established in Topic 1, web servers handle HTTP request-response cycles. They are optimized for:
- **Fast responses** (milliseconds to low seconds)
- **High concurrency** (many simultaneous lightweight connections)
- **Stateless request handling**

Heavy compute tasks are the **opposite** of this:
- **Slow** (seconds, minutes, sometimes hours)
- **Resource-intensive** (CPU, RAM, GPU)
- **Stateful** (need to track progress, store intermediate results)

Trying to run heavy compute **inside** the web server process is the root cause of all the problems seen in Topic 1.

### The Solution: Architectural Separation

The key insight is to **split responsibilities** across two distinct tiers:

```
┌─────────────────────────────────┐      ┌──────────────────────────────────┐
│         WEB SERVER TIER         │      │        COMPUTE / WORKER TIER     │
│                                 │      │                                  │
│  • Handle HTTP requests         │      │  • Execute heavy tasks           │
│  • Serve HTML / JSON / files    │      │  • CPU/GPU-intensive work        │
│  • Business logic (fast paths)  │      │  • Long-running processing       │
│  • Authentication / sessions    │      │  • External API calls (bulk)     │
│  • Input validation             │      │  • Batch operations              │
│                                 │      │                                  │
│  Optimized for: Speed & I/O     │      │  Optimized for: Throughput       │
└────────────────┬────────────────┘      └──────────────────────────────────┘
                 │   enqueue task (fast)              ▲
                 └────────────────────────────────────┘
                          via Task Queue / Message Broker
```

### What Gets Separated?

| Web Server Handles | Worker Tier Handles |
|---|---|
| Serving HTML pages & templates | Image / video processing |
| REST API responses | Machine learning inference |
| User authentication | Sending bulk emails / SMS |
| File uploads (receiving) | PDF generation |
| Database reads (fast queries) | Large report generation |
| Session management | Web scraping / crawling |
| Input validation | Data aggregation / ETL pipelines |

### Offloading to Specialized Compute Nodes

Workers (compute nodes) are **separate processes** — they can run:
- On the **same machine** as the web server (simple setup, shares resources)
- On **separate machines** (dedicated compute servers, GPU machines)
- On **cloud infrastructure** (auto-scaling worker pools, spot instances)

This separation means you can run workers on **hardware optimized for compute** (high RAM, GPU, many CPU cores) while the web server runs on **hardware optimized for network I/O** (many small cores, high network bandwidth).

### Independent Scaling

One of the biggest architectural wins of separation is **independent scaling**:

```
Traffic spike (many users):          Heavy compute spike (many uploads):
                                      
Web servers:  [S1][S2][S3][S4]        Web servers:  [S1][S2]
Workers:      [W1]                    Workers:      [W1][W2][W3][W4][W5]

→ Scale web tier horizontally         → Scale worker tier horizontally
  without touching workers              without touching web servers
```

- During a **traffic spike**: scale web servers up, workers stay the same.
- During a **compute spike** (e.g., many uploads): scale workers up, web servers stay the same.
- Each tier scales to its own bottleneck independently — **cost-efficient and precise**.

---

## 2.2 Goals of Asynchronous Task Frameworks

An async task framework provides the infrastructure to:

### Goal 1: Task Definition

A way to **declare** a unit of work — a function or callable — as a "task" that can be dispatched for background execution.

```python
# Example with Celery
@celery.task
def send_welcome_email(user_id):
    user = User.query.get(user_id)
    send_email(user.email, "Welcome!")
```

The framework needs to know:
- **What** to run (the function)
- **What arguments** to pass (serialized and sent via the queue)
- **Which queue** to send it to (routing)

### Goal 2: Dispatch Mechanism ("Fire and Forget" / Execute Later)

A way to **submit** a task for execution without waiting for it to complete.

```python
# Dispatch immediately — returns instantly, task runs in background
send_welcome_email.delay(user_id=42)

# Schedule for later execution
send_welcome_email.apply_async(user_id=42, countdown=300)  # run in 5 minutes

# Schedule at a specific time
send_welcome_email.apply_async(user_id=42, eta=datetime(2026, 9, 5, 9, 0))
```

- **"Fire and forget"**: Dispatch the task and move on — no waiting.
- The web request handler returns `HTTP 202 Accepted` immediately.
- The task executes asynchronously on a worker, independently of the client connection.

### Goal 3: Asynchronous Execution, Completion, and Status Tracking

The framework must provide a way to:
- **Execute** tasks on workers concurrently
- **Report completion** (success or failure)
- **Track status** (pending → started → success/failure)
- **Store results** (so the caller can retrieve the output later)

```python
# Get a handle to the task
result = send_welcome_email.delay(user_id=42)

# Check status later (polling)
result.status    # "PENDING" | "STARTED" | "SUCCESS" | "FAILURE"
result.ready()   # True if done
result.get()     # Blocking wait for result (use carefully!)
```

**Lifecycle of an async task:**

```
[DISPATCH] → [PENDING] → [STARTED] → [SUCCESS]
                                    ↘ [FAILURE] → [RETRY] → ...
```

---

## 2.3 Decision Framework: When to Use vs. When NOT to Use Async Tasks

Not every task should be made asynchronous. The key question is:

> **"Does the user's immediate response depend on the result of this operation?"**

### ✅ When TO Use Async Tasks

Use async tasks when the **user does not need the result right away** — when you can return a confirmation immediately and complete the work later.

| Use Case | Why Async? |
|---|---|
| Sending welcome/confirmation emails | User doesn't need to wait for email delivery |
| Generating a PDF report | User can be notified when ready to download |
| Processing an uploaded video | Encoding takes minutes; show "processing..." |
| Sending push notifications to followers | Fan-out to thousands — user doesn't wait |
| Running a machine learning model | Inference can take seconds/minutes |
| Resizing / compressing uploaded images | Store thumbnail URL when done |
| Scheduled jobs (daily digest emails) | Inherently time-based, not request-response |
| Updating a search index after content edit | Index update can lag slightly |

**Pattern:** Submit task → return `202 Accepted` or "Your request is being processed" → notify user when done (polling endpoint, WebSocket push, email, notification).

### ❌ When NOT to Use Async Tasks

Do **not** use async tasks when the **response depends directly on the task result** — when the user is actively waiting for the answer.

| Use Case | Why NOT Async? |
|---|---|
| Fetching live stock price for display | User needs the current value to see the page |
| Login / authentication check | Must know if credentials are valid right now |
| Form validation against a database | Must show errors before submission succeeds |
| Returning paginated search results | User is waiting for search output |
| Payment processing status | Must know success/failure to show next page |

Using async in these cases **adds complexity with no benefit** — you'd dispatch a task, immediately poll for the result, and wait — which is just synchronous execution with extra steps and added latency.

### The Critical Clarification: Backend vs. Frontend Async

A very common source of confusion:

| Term | What It Means | Example |
|---|---|---|
| **Backend Async Tasks** | Heavy operations offloaded to worker processes via a task queue | Celery, RQ, background jobs |
| **Frontend Async (JavaScript)** | Non-blocking UI updates using `async/await`, Promises, AJAX | `fetch()`, `axios`, reactive state updates |

These are **completely different concepts** that happen to share the word "asynchronous":
- **Frontend async** is about keeping the browser UI responsive while waiting for an HTTP response (which completes in seconds).
- **Backend async tasks** are about offloading work that takes too long to complete within a single HTTP request-response cycle.

```
Frontend async:
User clicks "Submit"
    │
    ▼ (non-blocking JS fetch)
Browser UI stays responsive ←──────────────────────── HTTP response arrives (~200ms)

Backend async task:
HTTP request arrives at server
    │
    ▼ Task enqueued (fast, ~5ms)
HTTP 202 Accepted ──────────────────────────────────► Response to browser
    │
    ▼ (separately, on worker)
[====== Heavy task runs for 3 minutes ======] ──► Result stored / notification sent
```

---

## 2.4 Essential Architectural Components

An async task system requires **two fundamental subsystems** working together:

### Component 1: Messaging / Communication System

Responsible for **transmitting task definitions and their arguments** from the dispatcher (web server) to the executor (worker).

#### Sub-components:

**Message Queue**
- A **FIFO buffer** that holds task messages until a worker picks them up.
- Decouples producer (web server) from consumer (worker) in time.
- Provides **durability** — if the worker crashes, messages remain in the queue.
- Enables **backpressure** — queue depth signals when the system is overloaded.

**Message Broker**
- Software that manages the message queue(s).
- Handles routing, persistence, delivery guarantees, and consumer management.
- Examples: **RabbitMQ**, **Redis**, **Apache Kafka**, **AWS SQS**

**Result Backend**
- A **separate storage** for task results once they complete.
- Workers write results here; callers read results from here.
- Must be separate from the broker because result lookup patterns differ from message queue patterns.
- Examples: **Redis**, **Memcached**, **Database (SQLAlchemy)**, **Amazon S3**

```
Web Server                Message Broker           Result Backend
──────────                ──────────────           ──────────────
Dispatch task ──────────► [Queue: tasks]           [Redis / DB]
                                  │                     ▲
                                  ▼                     │
                          Worker picks up task          │
                          Executes task ───────────────►│ stores result
```

### Component 2: Execution System (Workers)

Responsible for **actually running** the task functions.

Workers can be implemented using different concurrency primitives:

| Execution Model | Description | Best For |
|---|---|---|
| **Threads** | OS threads within a process | I/O-bound tasks, simple parallelism |
| **Processes** | Separate OS processes (no GIL) | CPU-bound tasks in Python |
| **Coroutines** | Cooperative multitasking (`async/await`) | High-concurrency I/O tasks |
| **Greenlets** | Lightweight cooperative threads (gevent) | High-concurrency with monkey-patching |
| **Separate Runtimes** | Workers on entirely different machines | Large-scale distributed systems |

Workers typically run in a **worker pool**:
```
                         ┌─────────────┐
                         │  Worker 1   │ ← picks task from queue
Message Broker ──────────│  Worker 2   │ ← picks task from queue
   [Queue]               │  Worker 3   │ ← picks task from queue
                         │  Worker N   │ ← picks task from queue
                         └─────────────┘
```

Multiple workers process tasks **concurrently** — the queue provides work; workers consume it.

### Python Ecosystem Example: Celery

**Celery** is the most widely used async task framework for Python. It provides both components:

```
Your Flask/Django App
       │
       │ .delay() / .apply_async()
       ▼
   [Celery] ──► Message Broker (RabbitMQ or Redis) ──► Celery Workers
                                                              │
                                                              ▼
                                                      Result Backend (Redis/DB)
```

- **Celery** handles task definition, serialization, dispatch, routing, retries, and result storage.
- **Message Broker** (RabbitMQ / Redis) handles the actual queue storage and delivery.
- **Result Backend** (Redis / DB) stores return values and task states.
- Your application only needs to call `.delay()` — Celery handles everything else.

> Celery is covered in detail in **Topic 5**. The next topics (3 & 4) build the foundations of how messaging and queues actually work.
