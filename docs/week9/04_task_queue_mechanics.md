# Topic 4: Task Queue Mechanics & Asynchronous Execution



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 4: Task Queue Mechanics & Asynchronous Execution**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 4.1 Task Queue Architecture

### The Full Pipeline

A task queue system has a well-defined flow from the moment a task is dispatched to the moment its result is stored:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TASK QUEUE PIPELINE                                  │
│                                                                             │
│  HTTP Request                                                               │
│      │                                                                      │
│      ▼                                                                      │
│  [Request Handler]  ──enqueue task──►  [Queue Manager / Broker]            │
│      │                                         │                           │
│      ▼                                         ▼                           │
│  HTTP 202 Accepted             [Worker pulls task from queue]               │
│  (returns immediately)                         │                           │
│                                                ▼                           │
│                                     [Worker executes task]                 │
│                                                │                           │
│                                                ▼                           │
│                                     [Result Backend / Storage]             │
│                                                │                           │
│                                                ▼                           │
│                                  [Notify user / poll endpoint]             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Step-by-step breakdown:**

| Step | Actor | Action | Latency |
|---|---|---|---|
| 1 | Request Handler | Receives HTTP request, validates input | ~ms |
| 2 | Request Handler | Serializes task + args → sends to broker | ~1–5ms |
| 3 | Request Handler | Returns `HTTP 202 Accepted` to client | ~ms |
| 4 | Queue Manager | Stores message in queue (durably) | ~ms |
| 5 | Worker | Polls or receives task from broker | ~ms to seconds |
| 6 | Worker | Deserializes task payload, executes function | seconds to minutes |
| 7 | Worker | Writes result / status to Result Backend | ~ms |
| 8 | Client | Polls status endpoint OR receives push notification | as needed |

The key insight: **Steps 1–3 are in the critical path of the HTTP response** — they must be fast. Steps 5–8 happen entirely off the critical path, decoupled from the client.

### FIFO Ordering vs. Priority Queueing

**Default: FIFO (First In, First Out)**
```
Enqueue order:  Task A → Task B → Task C → Task D
Dequeue order:  Task A → Task B → Task C → Task D
                (same order, strictly sequential per queue)
```
- Simple, fair, predictable.
- Most task queues default to FIFO.
- Tasks are processed in submission order.

**Priority Queueing**
```
Enqueue:  Task A (priority=1) → Task B (priority=5) → Task C (priority=3)
Dequeue:  Task B (priority=5) → Task C (priority=3) → Task A (priority=1)
                (highest priority processed first)
```
- Use when some tasks are more urgent than others.
- Example: Premium user's task vs. free-tier user's task.
- Implemented via: multiple queues (high/medium/low), or broker-level priority (RabbitMQ priority queues, Redis sorted sets).

**Common pattern — Multiple Queues by Priority:**
```
[high_priority_queue]   ──► Worker (checks this first)
[normal_queue]          ──► Worker (checks if high is empty)
[low_priority_queue]    ──► Worker (checks if others are empty)
```

### Asynchronous SLA: No Strict Real-Time Guarantees

A fundamental property of async task queues is that they provide **no strict real-time latency guarantees**:

- Tasks are executed when a worker is **available** — not necessarily immediately.
- If workers are saturated, tasks wait in the queue. Queue depth grows.
- The system guarantees **eventual completion** (assuming workers are running), not **immediate** completion.
- This is an intentional trade-off: you accept variable latency in exchange for reliability and decoupling.

> **SLA (Service Level Agreement) for async tasks** is typically expressed as:
> "Tasks will be completed within X minutes under normal load conditions"
> — NOT "Tasks will complete within 500ms" (that's a synchronous SLA).

---

## 4.2 Execution Models & Reliability Guarantees

### Language-Level Concurrency Mechanisms

Workers can use different concurrency primitives depending on the language and task type:

**Python `asyncio` (Coroutines)**
```python
import asyncio

async def process_task(task_id):
    data = await fetch_from_db(task_id)   # non-blocking I/O wait
    result = await run_ml_model(data)      # non-blocking compute
    await store_result(result)

asyncio.run(process_task(42))
```
- Single-threaded, event-loop-based cooperative multitasking.
- Excellent for **I/O-bound tasks** (database calls, API calls, file I/O).
- `await` yields control back to the event loop while waiting — other coroutines run.
- **Not suitable** for CPU-bound tasks (blocks the event loop).

**JavaScript `async`/`await` (Promises)**
```javascript
async function processTask(taskId) {
    const data = await fetchFromDB(taskId);     // non-blocking
    const result = await runModel(data);         // non-blocking
    await storeResult(result);
}
```
- JavaScript's single-threaded model with an event loop (Node.js).
- Same pattern as Python asyncio — cooperative, I/O-optimized.
- CPU-bound work must be offloaded to Worker Threads or child processes.

**Python `threading` (OS Threads)**
- Multiple threads in the same process — good for I/O-bound parallel tasks.
- Limited by the GIL for CPU-bound work (see Topic 1).

**Python `multiprocessing` (OS Processes)**
- Separate Python processes — true CPU parallelism (no GIL).
- Each process has its own memory space — no shared state (communicate via IPC/queues).
- Best for: CPU-intensive tasks (ML inference, image processing, data crunching).

**Celery Worker Concurrency Models:**
```
celery worker --concurrency=4 --pool=prefork      # 4 child processes (CPU-bound)
celery worker --concurrency=100 --pool=gevent     # 100 greenlets (I/O-bound)
celery worker --concurrency=100 --pool=eventlet   # 100 coroutines (I/O-bound)
celery worker --concurrency=4 --pool=threads      # 4 OS threads
```

### Reliability Guarantees

A task queue framework must answer: **"What happens if something goes wrong?"**

**Guarantee 1: At-Least-Once Delivery**
- Every message is delivered to a consumer **at least once**.
- If a worker crashes mid-task, the message is **re-queued** and delivered to another worker.
- Implication: **tasks may execute more than once** — tasks should be **idempotent** (safe to run twice).

```
Worker picks task ──► starts executing ──► worker crashes
                                                  │
Broker: "ack not received" ──► re-queues task ──► another worker picks it up
```

**Guarantee 2: At-Most-Once Delivery**
- Message is delivered at most once — if the worker crashes, the task is **lost** (not retried).
- Simpler, but risks data loss.
- Use when duplicate execution is worse than missed execution (e.g., financial debits).

**Guarantee 3: Exactly-Once Delivery**
- The holy grail — delivered exactly once, even in failure scenarios.
- Very hard to achieve in distributed systems; requires distributed transactions or idempotency keys.
- Most practical systems achieve this through **idempotent tasks + at-least-once delivery**.

**Automatic Retries**

Frameworks like Celery provide built-in retry logic:
```python
@celery.task(bind=True, max_retries=3, default_retry_delay=60)
def send_email(self, user_id):
    try:
        actually_send_email(user_id)
    except EmailServiceDown as exc:
        raise self.retry(exc=exc, countdown=60)  # retry in 60 seconds
```

Retry strategies:
- **Fixed delay**: Retry every N seconds.
- **Exponential backoff**: Retry after 1s, 2s, 4s, 8s, 16s... (reduces thundering herd).
- **Max retries**: Give up after N attempts → move to **Dead Letter Queue (DLQ)**.

---

## 4.3 Foundational Principles of Task Queues

### Principle 1: Enqueue Latency Must Be Much Less Than Execution Latency

**The fundamental invariant:**

$$T_{\text{push}} \ll T_{\text{exec}}$$

Where:
- $T_{\text{push}}$ = time to enqueue a task (send message to broker)
- $T_{\text{exec}}$ = time to execute the task (run on worker)

**Why this must hold:**

The entire value proposition of a task queue is that the web server can respond to the client **before** the task completes. If enqueueing itself is slow, the web server blocks on the enqueue step — defeating the purpose.

```
Good (T_push << T_exec):
Web server: [validate input][enqueue: 2ms][return 202]  ← fast!
Worker:     [========= execute task: 5 minutes =========]

Bad (T_push ≈ T_exec):
Web server: [validate input][enqueue: 30s...][return 202]  ← defeats the purpose!
```

**Practical implications:**
- Broker must be **low-latency** and **co-located** (same datacenter, ideally same network).
- Message payloads must be **small** — don't send large files through the queue; send a reference (file path, S3 URL, DB ID) and let the worker fetch it.
- The broker should **never be the bottleneck** in the enqueue path.

```
❌ Bad: enqueue(image_binary_data)   # large payload, slow
✅ Good: enqueue(image_id=42)        # tiny payload, worker fetches from storage
```

### Principle 2: Worker Capacity Equilibrium

**The fundamental stability condition:**

$$\text{Consumption Rate} \geq \text{Arrival Rate} \quad \text{(long-term average)}$$

Where:
- **Arrival Rate** = tasks enqueued per unit time (driven by user traffic)
- **Consumption Rate** = tasks completed per unit time (driven by worker count × task speed)

**What happens when the rates diverge:**

```
Case A: Consumption ≥ Arrival (Stable ✅)
Queue depth: [■■■■] → [■■■] → [■■] → [■] → [] ← drains over time

Case B: Consumption < Arrival (Unstable ❌)
Queue depth: [■■] → [■■■■] → [■■■■■■■■] → [■■■■■■■■■■■■■■■■■] → OVERFLOW
```

**Consequences of queue overflow:**
- New tasks are **rejected** (broker returns error to producer).
- Tasks are **dropped** (lost messages).
- Broker runs **out of memory** (RAM exhaustion).
- System eventually crashes or becomes unresponsive.

**How to maintain equilibrium:**
1. **Add more workers** (horizontal scaling) — increase consumption rate.
2. **Optimize task execution** — make individual tasks faster.
3. **Rate-limit producers** — slow down enqueue rate (shed load at the source).
4. **Use multiple queues** — priority queues ensure critical tasks are processed first.
5. **Auto-scale workers** based on queue depth metrics (e.g., AWS Auto Scaling + SQS depth).

---

## 4.4 Common Distributed Pitfalls

### Pitfall 1: Deadlocks & Broker Unavailability

**Scenario:** The message broker itself goes down (crash, network partition, maintenance).

**Impact:**
- Producers cannot enqueue — task dispatch calls **block** (waiting for broker reconnection) or **fail** (throw exceptions).
- Workers cannot dequeue — idle, no work being processed.
- Entire async task system is offline until broker recovers.

**Block vs. Drop Decision:**

| Strategy | Behaviour | Use When |
|---|---|---|
| **Block** | Producer waits (with timeout) for broker to come back | Task must not be lost; brief outages expected |
| **Drop** | Producer discards task, logs error, returns error response | Low-value tasks where loss is acceptable |
| **Circuit Breaker** | After N failures, stop attempting (fail fast) for T seconds | Protect producer from cascading failures |
| **Local Fallback Queue** | Buffer tasks locally (in memory or DB) while broker is down | High-reliability systems; complex to implement |

**Deadlock scenario in Celery:**
```
Task A enqueues Task B (child task) and waits for result.
But the worker pool is full processing Task A's siblings.
Task B can never start → Task A waits forever → DEADLOCK.
```
Solution: Use `task.apply_async()` with `link` callbacks instead of `.get()` inside tasks. Never call `.get()` from within a task!

### Pitfall 2: Buffer Sizing, Memory Limits, and Queue Overflow

**In-Memory Brokers (Redis default):**
- Redis stores all queue data in RAM.
- Queue can grow unboundedly until Redis hits its `maxmemory` limit.
- When limit is hit, Redis may **evict messages** (data loss!) depending on `maxmemory-policy`.

```
Dangerous Redis config for queues:
maxmemory 512mb
maxmemory-policy allkeys-lru   ← will evict queue messages to free RAM!

Safe config:
maxmemory 512mb
maxmemory-policy noeviction    ← broker returns error instead of losing messages
```

**RabbitMQ:**
- Queues have configurable `max-length` and `max-length-bytes`.
- When limits are hit: either **reject new messages** (producer gets error) or **drop oldest messages** (head-of-queue drop).

**Best practices:**
- Set **explicit queue length limits** — never allow unbounded growth.
- Monitor queue depth continuously — alert when depth exceeds thresholds.
- Configure **Dead Letter Exchanges (DLX)** — overflow messages go to a DLQ instead of being silently dropped.
- Size your worker fleet so queue depth stays near zero under normal load (reserve capacity for spikes).

---

## 4.5 Task Queue Delivery Models

### Push Queue

The **broker actively delivers** tasks to workers as soon as they are enqueued.

```
Producer ──enqueue──► [Broker]
                          │ immediately pushes
                          ▼
                       Worker (receives task via active connection / callback)
```

**Characteristics:**
- Near real-time delivery — task reaches a worker in milliseconds.
- Worker maintains a **persistent connection** to the broker (TCP connection, AMQP channel).
- Broker knows which workers are connected and available.
- Worker registers with the broker as a consumer — broker **pushes** tasks when available.

**Use cases:**
- Transactional emails (send immediately after user action)
- Real-time feed updates
- Live notification dispatch
- Any task where low latency between enqueue and execution matters

**Examples:** RabbitMQ (AMQP consumer), Celery with RabbitMQ backend.

---

### Pull Queue

Workers **actively poll** the broker at regular intervals to check for and claim tasks.

```
Producer ──enqueue──► [Broker Queue]

                           ↑↑↑ (workers periodically check)

Worker 1 ──poll──► "any tasks?" ──► picks up task1
Worker 2 ──poll──► "any tasks?" ──► picks up task2
Worker 3 ──poll──► "any tasks?" ──► "queue empty, wait"
```

**Characteristics:**
- Workers are **stateless** with respect to the broker — no persistent connection needed.
- Workers control the **rate** at which they consume tasks.
- Natural **backpressure**: workers only request tasks when they're ready for more work.
- Slight latency between enqueue and execution (depends on poll interval).

**Use cases:**
- Batch job processing (leaderboard recalculations, report generation)
- Log aggregation and batch writes
- ETL pipelines
- Any workload where slight delay is acceptable, and batch collection is beneficial

**Examples:** AWS SQS (workers poll SQS API), Google AppEngine Task Queue.

---

### Pull Mechanisms: Short Polling vs. Long Polling

Since pull queues require workers to actively check for work, **how** they poll matters:

#### Short Polling (Naive)
```
while True:
    tasks = queue.get_tasks()   # check immediately
    if tasks:
        process(tasks)
    else:
        time.sleep(poll_interval)  # wait, then check again
```

```
Timeline:
Worker: poll ──► empty │ poll ──► empty │ poll ──► TASK! │ poll ──► empty │ ...
         T=0           T=1             T=2              T=3
```

**Characteristics:**
- Simple to implement.
- Creates constant traffic to the broker even when queue is empty.
- **Wasted CPU and network** — most polls return empty during low-traffic periods.
- **Latency** = up to `poll_interval` (e.g., if polling every 5s, task waits up to 5s before pickup).
- Higher poll frequency = lower latency but more wasted requests.

#### Long Polling
```
while True:
    # Server HOLDS the connection open for up to 20 seconds
    # Returns immediately IF a task arrives during that window
    tasks = queue.get_tasks(wait_seconds=20)
    if tasks:
        process(tasks)
    # Loop immediately — no sleep needed, server already waited
```

```
Timeline:
Worker: long-poll ────────── (server holds 20s) ──── TASK ARRIVES! ──► Process
         T=0                                    T=7  (immediate response)
```

**Characteristics:**
- The broker **holds the HTTP/TCP connection open** until a task arrives or timeout expires.
- When a task is enqueued, the waiting worker receives it **immediately**.
- Far **fewer wasted requests** — one long-poll per worker per idle period vs. constant polling.
- Effectively combines the simplicity of pull with near-real-time delivery of push.
- **Latency** ≈ milliseconds (task arrives → connection unblocks immediately).

**Comparison:**

| | Short Polling | Long Polling |
|---|---|---|
| Implementation | Simple | Moderate |
| Broker connections | Many (frequent, short) | Few (infrequent, long-lived) |
| Network overhead | High (empty responses) | Low |
| Idle latency | Up to `poll_interval` | Near-zero (immediate on arrival) |
| CPU overhead | High | Low |
| **Best for** | Very low-traffic, simple setup | Medium-to-high scale, efficiency matters |

---

## 4.6 Industry Implementations & Ecosystem

### Cloud Managed / High-End (Fully Managed Services)

| Service | Provider | Key Features |
|---|---|---|
| **Google AppEngine Task Queue** | Google Cloud | Push/Pull queues, built-in scaling, tight GAE integration |
| **AWS SQS (Simple Queue Service)** | Amazon | Managed FIFO & standard queues, long polling, dead letter queues, auto-scales to any volume |
| **AWS SQS + Lambda** | Amazon | Serverless workers — SQS event triggers Lambda functions (zero server management) |
| **Tencent Cloud CMQ** | Tencent | Message Queue service, similar to SQS, Asia-Pacific optimized |
| **Azure Service Bus** | Microsoft | Enterprise messaging, topics/subscriptions (pub/sub), AMQP support |
| **Google Cloud Pub/Sub** | Google | Global pub/sub, push/pull, at-least-once delivery, very high throughput |

**Trade-offs of managed services:**
- ✅ Zero infrastructure management (no broker servers to maintain)
- ✅ Automatic scaling to any volume
- ✅ Built-in HA, replication, durability
- ❌ Vendor lock-in
- ❌ Cost at scale (per-message pricing)
- ❌ Less flexibility in routing/configuration vs. self-hosted RabbitMQ

---

### Python / General Open Source

| Library | Broker Support | Concurrency | Best For |
|---|---|---|---|
| **Celery** | RabbitMQ, Redis, SQS, others | Prefork, Gevent, Eventlet, Threads | Full-featured production task queues; Flask/Django |
| **RQ (Redis Queue)** | Redis only | Forked processes | Simplicity; Redis already in stack; small-to-medium scale |
| **Huey** | Redis, SQLite, in-memory | Threads, Greenlets | Lightweight Celery alternative; simpler config |
| **Django-Carrot** | RabbitMQ (AMQP) | Threads | Django-native, simpler than Celery for Django projects |
| **Dramatiq** | Redis, RabbitMQ | Threads, Processes | Modern Celery alternative; simpler API, better defaults |
| **arq** | Redis | asyncio coroutines | Async-first worker; best for async Python apps |

**Quick comparison — Celery vs. RQ:**

| | Celery | RQ (Redis Queue) |
|---|---|---|
| Broker support | RabbitMQ, Redis, SQS, many more | Redis only |
| Setup complexity | ⭐⭐⭐ Complex | ⭐ Simple |
| Features | Full-featured (chains, chords, groups, eta, rate-limiting) | Basic (enqueue, retry) |
| Performance | High | High (simpler overhead) |
| Monitoring | Flower (web UI) | RQ Dashboard |
| Production readiness | Battle-tested at scale | Good for small-medium scale |
| **Best for** | Large projects needing full control | Quick setup, Redis-only stacks |

**Celery** is the de facto standard for Python async tasks and is covered in detail in **Topic 5**.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

