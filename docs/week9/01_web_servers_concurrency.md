# Topic 1: Web Servers & Concurrency Foundations



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 1: Web Servers & Concurrency Foundations**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 1.1 How Web Servers Work

A web server, at its simplest, is a program that **listens on a network port** (typically **port 80** for HTTP, **port 443** for HTTPS) for incoming client connections and sends back responses.

### Minimal HTTP Server Loop

Every web server — no matter how complex — is built on this fundamental cycle:

```
1. LISTEN   — Open a socket, bind to port 80, wait for connections
2. ACCEPT   — A client connects; accept the TCP connection
3. READ     — Read the raw bytes of the HTTP request from the socket
4. PARSE    — Parse the HTTP request: method (GET/POST), path, headers, body
5. PROCESS  — Run application logic to generate the appropriate response
6. RESPOND  — Serialize & send back HTTP response: status line + headers + body
7. CLOSE    — Close the connection (or keep-alive for HTTP/1.1 persistent connections)
8. REPEAT   — Go back to step 1 (or 2 if already listening)
```

### Simplest Implementation (Single-Threaded, Synchronous)

In the most naive implementation, **all 8 steps happen sequentially in a single loop** on a single thread:

```python
while True:
    conn = socket.accept()       # Step 2
    request = conn.read()        # Step 3-4
    response = handle(request)   # Step 5
    conn.write(response)         # Step 6
    conn.close()                 # Step 7
```

- After finishing request A, the server loops back and handles request B.
- While handling A, **no other request can be accepted or processed**.
- This is a **single-threaded synchronous** (blocking) server.

---

## 1.2 Connection Handling Modes in Web Frameworks (Flask)

Flask's built-in development server exposes a `threaded` parameter in `app.run()` that controls how it handles multiple simultaneous requests.

### Non-Threaded Mode (`threaded=False`)

```python
app.run(threaded=False)
```

- Processes **one request at a time** — all others wait in the OS accept queue.
- If request A is slow (e.g., 10 seconds), request B waits the full 10 seconds before even starting.
- **Use when:** Simple single-user debugging, no concurrency required.

### Threaded Mode (`threaded=True`)

```python
app.run(threaded=True)   # Flask default
```

- Spawns a **new OS thread per incoming request**.
- Each request runs independently; a slow request on Thread 1 does not delay Thread 2.
- **Use when:** Development with multiple simultaneous clients, testing concurrent behaviour.

### Summary Table

| Mode | Behaviour | Blocked by slow request? |
|---|---|---|
| `threaded=False` | Sequential, one at a time | ✅ Yes — all clients wait |
| `threaded=True` *(default)* | New thread per request | ❌ No — threads are isolated |

> ⚠️ **Flask's built-in server is NOT production-ready.** In production, use a proper **WSGI server** like:
> - **Gunicorn** — pre-fork worker model (multiple processes)
> - **uWSGI** — highly configurable, supports multiple concurrency modes
> - **Waitress** — pure Python, good for Windows
>
> These manage worker pools, process recycling, and concurrency far more robustly.

---

## 1.3 Threaded Web Server Architecture

### The Request-per-Thread Model

The most common simple concurrency approach: **spin up one OS thread per incoming request**.

```
Incoming Requests        Threads              Responses
─────────────────        ───────              ─────────
Client A ──────────────► Thread 1 ──────────► Response A
Client B ──────────────► Thread 2 ──────────► Response B
Client C ──────────────► Thread 3 ──────────► Response C
```

Each thread independently owns its entire request lifecycle — from parsing the HTTP bytes to sending the response. Threads share the process's memory space but have their own **call stack**.

### Limitations of Request-per-Thread

#### 1. Memory / Resource Consumption
- Each OS thread requires its own **stack memory** (~1–8 MB per thread depending on OS).
- 1000 concurrent requests = 1000 threads = potentially **~8 GB of RAM** just for thread stacks.
- CPU has overhead for **context switching** — when the OS switches between threads, it must save/restore register state.

#### 2. OS-Controlled Scheduling
- Thread scheduling is done by the **OS kernel**, not the application.
- The app cannot say "prioritize Thread 3 over Thread 1" — the OS decides.
- This introduces **non-determinism** in execution order.
- Context switch overhead adds **latency** especially when many threads are competing.

#### 3. Thread Pool Exhaustion
- In practice, servers use a **fixed-size thread pool** (e.g., 100 threads max) rather than unbounded spawning.
- When all threads are busy, new requests must **wait or be dropped**.
- Under bursty load, even a 200-thread pool can saturate instantly.

### Concurrency vs. Parallelism

These two terms are frequently conflated but are fundamentally different:

| Concept | What It Means | Hardware Needed |
|---|---|---|
| **Concurrency** | Multiple tasks are *in progress* simultaneously (interleaved/time-sliced) | Works on a **single core** |
| **Parallelism** | Multiple tasks are *physically executing* at the exact same instant | Requires **multiple CPU cores** |

**Key insight:**
- **Concurrency** is a *program structure* — about managing many things at once.
- **Parallelism** is a *hardware execution* — about doing many things at once.
- You can have concurrency **without** parallelism (single-core time-slicing).
- You can't have parallelism **without** concurrency.

```
Concurrency on 1 core (time-slicing):
Core 1: [─A─][─B─][─A─][─C─][─B─][─A─]   ← tasks interleaved

Parallelism on 3 cores:
Core 1: [────────── A ──────────]
Core 2: [────────── B ──────────]           ← tasks truly simultaneous
Core 3: [────────── C ──────────]
```

> **Python's GIL (Global Interpreter Lock):**
> CPython (the standard Python interpreter) has a GIL — a mutex that allows only **one thread to execute Python bytecode at a time**.
> - **CPU-bound tasks** (heavy computation): threads give concurrency but NOT true parallelism — GIL blocks simultaneous execution.
> - **I/O-bound tasks** (network, disk, database): the GIL is **released during I/O waits**, so threading is genuinely beneficial — other threads run while one waits for I/O.
>
> For true CPU parallelism in Python: use `multiprocessing` (separate processes, no GIL) or external workers.

---

## 1.4 The Blocking Server Problem

### What Is Blocking?

A server is **blocking** when it holds an operation (or the entire server) in a wait state while a task executes — preventing other work from proceeding.

### Client-Side Perspective

When a client's request triggers a long synchronous operation on the server:
- The **HTTP connection stays open** for the entire duration of the task.
- The browser/client is **stuck waiting** for a response.
- The user **cannot navigate away** without cancelling the request mid-execution.
- **Network timeouts** may fire before the task completes (e.g., Nginx default 60s timeout, some browsers timeout at 30s).
- The page appears **frozen / unresponsive** — terrible UX.

### Single-Threaded Blocking Server

```
Timeline:
─────────────────────────────────────────────────────────────
Client A: ══════════════ Long Task (30s) ═══════════════╗
Client B:                                               ║  [queued for 30s] ──► Response B
Client C:                                               ║                       [queued for 60s] ──► Response C
─────────────────────────────────────────────────────────────
```

- Client B and C **cannot even start** until Client A's task finishes.
- The server is **completely unavailable** for the duration of any slow request.
- A single rogue slow request degrades the **entire service** for all users.

### Threaded Server — Thread Isolation

```
Timeline:
─────────────────────────────────────────────────────────────
Client A → Thread 1: ══════════════ Long Task (30s) ═══════════════╗
Client B → Thread 2: ══ Short Task ══╗                              ║  ← Response B (immediately)
Client C → Thread 3: ══ Short Task ══╗                              ║  ← Response C (immediately)
─────────────────────────────────────────────────────────────
```

- Threading **isolates the block** — Client B and C are unaffected by Client A's long task.
- But Thread 1 is **occupied for 30 seconds**, holding memory and a thread slot.
- Under high load with many long-running requests: **thread pool exhausts**, server degrades.

---

## 1.5 Long-Running Tasks Case Study: Face Recognition on Photo Upload

### Scenario

A social photo-sharing platform. When a user uploads a photo, the server must:

| Stage | Task | Speed |
|---|---|---|
| **Stage 1** | Detect faces in uploaded photo | ⚡ Fast (~ms) |
| **Stage 2** | Run deep learning face recognition model | 🐢 Slow (~seconds to minutes) |
| **Stage 3** | Search recognized faces against user database | 🐢 Slow (DB query, large dataset) |
| **Stage 4** | Send push/email notifications to tagged users | 🐌 Variable (external service calls) |

The full pipeline could take **minutes** per photo.

---

### Failure Mode 1: Single-Threaded (Blocking) Server

```
User uploads photo
       │
       ▼
[Server: Stage 1 — Face Detection]
       │
       ▼
[Server: Stage 2 — Face Recognition Model]  ← 2 minutes...
       │
       ▼
[Server: Stage 3 — Database Search]         ← 30 seconds...
       │
       ▼
[Server: Stage 4 — Send Notifications]      ← 15 seconds...
       │
       ▼  ← User has been BLOCKED this ENTIRE time
[HTTP Response: "Upload complete!"]
```

**Problems:**
- User's browser is frozen for ~3 minutes.
- User cannot do anything else on the platform during this time.
- All other users are completely blocked for those 3 minutes.
- Any network hiccup → timeout → the whole task is lost.

---

### Failure Mode 2: Naive Threaded Server

```
100 users upload simultaneously
       │
       ▼
100 threads spawned — each running full 4-stage pipeline
       │
100 × [Face Recognition Model] running concurrently
       │
       ▼
Server RAM: [████████████████████████ EXHAUSTED]
OS starts swapping to disk → severe slowdown
       │
       ▼
Server crashes or becomes completely unresponsive
```

**Problems:**
- Threading isolates each user's block, but **resource exhaustion** remains.
- Face recognition is **CPU + memory intensive** — running 100 simultaneously is impractical.
- **Uncontrolled thread spawning**: no upper limit on threads = no upper limit on resource usage.
- Even with a fixed thread pool (say, 10 threads): 90 users wait anyway, pool fills instantly.
- Server **degradation under load** is unpredictable and catastrophic.

---

### Root Problem: Tight Coupling of HTTP Request & Background Computation

The core architectural mistake is that the **HTTP request-response cycle is used to control the lifetime of a background computation**.

```
❌ Wrong:  [HTTP Connection] ──drives──► [Heavy Computation]
           ↑ if connection drops, computation is lost
           ↑ user must wait for compute to finish
           ↑ thread tied up for the full duration

✅ Right:  [HTTP Connection] ──enqueues──► [Task Queue] ──executes──► [Worker Process]
           ↑ connection closes immediately after enqueue (~ms)
           ↑ user gets instant confirmation
           ↑ worker runs independently, can retry on failure
```

**The solution is architectural decoupling:**
- The web server's job is to **enqueue the task** (fast, ~milliseconds) and immediately return `202 Accepted`.
- A **separate worker process** picks up and executes the task independently of any client connection.
- The user can be notified of completion later via **webhooks, polling, or push notifications**.

> This is the core motivation for **Asynchronous Task Queues** — the subject of the rest of Week 9.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

