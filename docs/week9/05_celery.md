# Topic 5: Asynchronous Tasks with Celery



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 5: Asynchronous Tasks with Celery**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 5.1 Celery Core Architecture

### What is Celery?

**Celery** is an open-source, distributed task queue framework for Python. It is the de facto standard for executing background jobs in Python web applications (Flask, Django, FastAPI).

Celery's job is to:
1. **Define** tasks as Python functions decorated with `@celery.task`
2. **Serialize** task calls (function name + arguments) into messages
3. **Dispatch** those messages to a Message Broker
4. **Coordinate** workers that pick up and execute those messages
5. **Store** results and status in a Result Backend
6. **Provide** an API to check task status and retrieve results

Celery itself is the **orchestration layer** — it does NOT implement the message queue or the result store itself. It delegates those to pluggable backends.

---

### The Three-Component Architecture

Celery always involves **three distinct components** working together:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CELERY SYSTEM                                   │
│                                                                         │
│   ┌──────────────┐    enqueue    ┌───────────────┐    dequeue          │
│   │  Application │ ────────────► │ Message Broker│ ───────────►        │
│   │  (Flask/     │               │ (RabbitMQ /   │             │       │
│   │   Django)    │               │  Redis)       │             ▼       │
│   │              │               └───────────────┘      ┌───────────┐  │
│   │  task.delay()│                                       │  Celery   │  │
│   │              │                                       │  Worker   │  │
│   │  result.get()│◄────────────────────────────────────  │  (process)│  │
│   └──────────────┘    read result   ┌───────────────┐   └─────┬─────┘  │
│                                     │ Result Backend│◄────────┘        │
│                                     │ (Redis / DB)  │  store result    │
│                                     └───────────────┘                  │
└─────────────────────────────────────────────────────────────────────────┘
```

| Component | Role | Common Implementations |
|---|---|---|
| **Application** | Defines tasks, dispatches them | Your Flask / Django / FastAPI app |
| **Message Broker** | Stores and routes task messages (the queue) | RabbitMQ, Redis, AWS SQS |
| **Result Backend** | Stores task results and status | Redis, Memcached, SQLAlchemy DB, MongoDB |
| **Celery Worker** | Picks up tasks from broker, executes them | Celery worker process(es) |

---

### Why Broker and Result Backend Are Separate

A common question: *"If Redis can be both broker and result backend, why distinguish them?"*

Because they serve **fundamentally different access patterns**:

| | Message Broker | Result Backend |
|---|---|---|
| **Access pattern** | Sequential queue (FIFO push/pop) | Random key-value lookup by task ID |
| **Lifetime** | Message deleted after consumed | Result stored until retrieved (or TTL expires) |
| **Consumers** | One consumer per message | Many readers can check same result |
| **Operations** | `LPUSH` / `BRPOP` (enqueue/dequeue) | `SET result_<id>` / `GET result_<id>` |
| **Durability** | Must not lose messages | Results can be ephemeral (TTL) |

Using the same Redis instance for both is fine (common in dev/small-scale), but they are logically separate and can be pointed at different servers in production.

---

### Task Auto-Discovery Across Multiple Worker Instances

Celery workers are **stateless and symmetric** — any worker can handle any task:

```
             ┌─────────────────────────────────────────┐
             │          Message Broker                  │
             │   [task1] [task2] [task3] [task4] ...   │
             └────────────────┬────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         Worker 1         Worker 2         Worker 3
         picks task1      picks task2      picks task3
         executes         executes         executes
```

- All workers connect to the **same broker** and consume from the **same queue(s)**.
- Work is **automatically distributed** — no central coordinator assigning tasks.
- Workers **auto-discover** available tasks by importing the application's task modules on startup.
- Adding a new worker = **linear throughput increase** with zero code changes.

**Auto-discovery configuration:**
```python
# celery_app.py
from celery import Celery

app = Celery('myproject')
app.config_from_object('myproject.celeryconfig')

# Auto-discover tasks in all installed Django apps
app.autodiscover_tasks()

# OR explicitly list modules:
app.autodiscover_tasks(['myapp.tasks', 'otherapp.tasks'])
```

Celery scans each listed module for functions decorated with `@app.task` and registers them. Workers import these modules on startup — they know which tasks they can handle.

---

### Abstraction Layer Over Messaging Protocols

One of Celery's key design decisions: it **abstracts away** the underlying messaging protocol.

Your application code looks identical whether the broker is RabbitMQ (AMQP), Redis (LIST operations), or AWS SQS:

```python
# Application code — same regardless of broker
from myapp.tasks import send_email, process_image

send_email.delay(user_id=42)        # works with RabbitMQ
process_image.delay(image_id=99)    # works with Redis
                                    # works with SQS
                                    # switch broker by changing one config line
```

Switching brokers requires only a config change:
```python
# celeryconfig.py

# Using RabbitMQ
broker_url = 'amqp://guest:guest@localhost:5672//'

# Using Redis
broker_url = 'redis://localhost:6379/0'

# Using AWS SQS
broker_url = 'sqs://AWS_ACCESS_KEY:AWS_SECRET_KEY@'
```

This abstraction means you can:
- Start development with **Redis** (easy setup)
- Move to **RabbitMQ** in production (more durable/featureful)
- Move to **SQS** if migrating to AWS (managed, no broker servers)
— all with **zero changes to your task code**.

---

## Setting Up Celery: Minimal Working Example

### Project Structure
```
myproject/
├── app.py              ← Flask app
├── tasks.py            ← Celery task definitions
├── celeryconfig.py     ← Celery configuration
└── requirements.txt
```

### Step 1: Install Dependencies
```bash
pip install celery redis flask
# OR for RabbitMQ:
pip install celery[rabbitmq] flask
```

### Step 2: Create the Celery App
```python
# tasks.py
from celery import Celery
import time

# Create Celery instance
# First arg: name of the current module (for auto-naming tasks)
# broker: where to send task messages
# backend: where to store results
celery_app = Celery(
    'myproject',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

# Define a task
@celery_app.task
def add(x, y):
    time.sleep(5)   # simulate slow work
    return x + y

@celery_app.task
def send_welcome_email(user_id):
    # simulate email sending
    time.sleep(2)
    print(f"Email sent to user {user_id}")
    return f"email_sent:{user_id}"
```

### Step 3: Dispatch Tasks from Flask
```python
# app.py
from flask import Flask, jsonify
from tasks import add, send_welcome_email

app = Flask(__name__)

@app.route('/add')
def trigger_add():
    # dispatch task — returns IMMEDIATELY (does not wait for result)
    result = add.delay(4, 6)
    return jsonify({
        "status": "queued",
        "task_id": result.id   # use this to check status later
    })

@app.route('/register/<int:user_id>')
def register_user(user_id):
    send_welcome_email.delay(user_id)   # fire and forget
    return jsonify({"status": "registered", "message": "Welcome email queued"})

@app.route('/status/<task_id>')
def task_status(task_id):
    result = add.AsyncResult(task_id)
    return jsonify({
        "task_id": task_id,
        "status": result.status,       # PENDING / STARTED / SUCCESS / FAILURE
        "result": result.result if result.ready() else None
    })
```

### Step 4: Run the Worker
```bash
# Start Celery worker (in a separate terminal)
celery -A tasks worker --loglevel=info

# With concurrency (4 parallel processes):
celery -A tasks worker --loglevel=info --concurrency=4

# With specific pool (greenlets for I/O-bound):
celery -A tasks worker --loglevel=info --pool=gevent --concurrency=100
```

### Step 5: Full Interaction Flow
```
1. Browser: GET /add
2. Flask: add.delay(4, 6) → enqueues message → returns task_id instantly
3. Browser receives: {"status": "queued", "task_id": "abc-123"}

4. (5 seconds later, on worker)
   Worker: picks up task → executes add(4, 6) → stores result 10 → backend

5. Browser: GET /status/abc-123
6. Flask: AsyncResult("abc-123").status → "SUCCESS", .result → 10
7. Browser receives: {"status": "SUCCESS", "result": 10}
```

---

## 5.2 Operational Considerations & Challenges

### Challenge 1: Multiple Moving Parts

A production Celery deployment involves at minimum **4 separate running services**:

```
┌──────────────────────────────────────────────────────┐
│                PRODUCTION CELERY STACK               │
│                                                      │
│  1. Web Application (Flask/Django)  ← your app       │
│     └─ talks to: Broker, Result Backend              │
│                                                      │
│  2. Message Broker (RabbitMQ / Redis)                │
│     └─ must be: running, reachable, healthy          │
│                                                      │
│  3. Result Backend (Redis / DB)                      │
│     └─ must be: running, reachable, healthy          │
│                                                      │
│  4. Celery Worker(s) (1 to N processes)              │
│     └─ must be: running, consuming, healthy          │
│                                                      │
│  5. (Optional) Celery Beat — scheduler               │
│     └─ for periodic/cron tasks                       │
│                                                      │
│  6. (Optional) Flower — monitoring dashboard         │
└──────────────────────────────────────────────────────┘
```

Every one of these must be:
- Deployed and started (separate process/container)
- Kept alive (restart on crash — `systemd`, `supervisor`, `Docker restart policy`)
- Monitored for health
- Upgraded independently (version compatibility)
- Backed up (broker persistence, result backend data)

**A bug, crash, or misconfiguration in any one component can silently break the entire async system.**

---

### Challenge 2: Deployment, Configuration, and Monitoring Overhead

#### Deployment Complexity

In a simple Flask app, deployment = ship the app, run `gunicorn`. With Celery:

```
Without Celery:                    With Celery:
─────────────                      ────────────
Deploy Flask app                   Deploy Flask app
                                   Deploy Redis/RabbitMQ
                                   Deploy Celery workers (N instances)
                                   Deploy Celery Beat (if periodic tasks)
                                   Deploy Flower dashboard (optional)
                                   Configure all services to talk to each other
                                   Set up process supervision (systemd/supervisor)
                                   Set up log aggregation across all processes
```

**With Docker Compose (recommended approach):**
```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build: .
    command: gunicorn app:app -b 0.0.0.0:5000
    depends_on: [redis]

  worker:
    build: .
    command: celery -A tasks worker --loglevel=info --concurrency=4
    depends_on: [redis]

  beat:
    build: .
    command: celery -A tasks beat --loglevel=info
    depends_on: [redis]

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

  flower:
    build: .
    command: celery -A tasks flower --port=5555
    ports:
      - "5555:5555"
    depends_on: [redis]

volumes:
  redis_data:
```

#### Configuration

Celery has **extensive configuration options** — key ones to know:

```python
# celeryconfig.py

# Broker & Backend
broker_url = 'redis://localhost:6379/0'
result_backend = 'redis://localhost:6379/1'

# Serialization (how task args are encoded)
task_serializer = 'json'        # json (safer), pickle (Python-only)
result_serializer = 'json'
accept_content = ['json']

# Task behavior
task_acks_late = True           # ack AFTER task completes (safer — requeue on crash)
task_reject_on_worker_lost = True  # requeue if worker dies mid-task
task_time_limit = 300           # hard kill task after 5 minutes
task_soft_time_limit = 240      # raise SoftTimeLimitExceeded after 4 minutes (graceful)

# Result expiry
result_expires = 3600           # delete results from backend after 1 hour

# Worker
worker_prefetch_multiplier = 1  # worker takes 1 task at a time (fair distribution)
worker_max_tasks_per_child = 100  # restart worker process after 100 tasks (prevent memory leaks)

# Routing (send different tasks to different queues)
task_routes = {
    'tasks.send_email': {'queue': 'email'},
    'tasks.process_image': {'queue': 'heavy'},
}
```

#### Monitoring with Flower

**Flower** is the official Celery monitoring web UI:

```bash
pip install flower
celery -A tasks flower --port=5555
# Visit http://localhost:5555
```

Flower shows:
- Active workers and their status
- Tasks in progress, completed, failed
- Task success/failure rates
- Queue depths per queue
- Worker CPU and memory usage
- Task history and results

---

### Challenge 3: Practical Constraints & Considerations

#### Task Idempotency

Because Celery uses **at-least-once delivery**, a task may execute more than once (worker crash + redeliver). Tasks must be **idempotent** — running them multiple times must be safe.

```python
# ❌ NOT idempotent — may charge user twice if task runs twice
@celery_app.task
def charge_user(user_id, amount):
    payment_gateway.charge(user_id, amount)

# ✅ Idempotent — uses idempotency key to prevent duplicate charges
@celery_app.task
def charge_user(user_id, amount, idempotency_key):
    if already_processed(idempotency_key):
        return "already_done"
    payment_gateway.charge(user_id, amount, key=idempotency_key)
    mark_processed(idempotency_key)
```

#### Never Call `.get()` Inside a Task (Deadlock Risk)

```python
# ❌ DANGEROUS — can deadlock
@celery_app.task
def parent_task():
    child = child_task.delay()
    return child.get()   # blocks waiting for child
                         # but child can't start if worker pool is full with parent tasks!

# ✅ SAFE — use callbacks / chains instead
from celery import chain

chain(parent_task.s(), child_task.s())()
```

#### Celery Canvas — Task Composition Primitives

Celery provides powerful primitives to compose tasks into workflows:

```python
from celery import chain, group, chord

# chain: tasks run in sequence, output of one → input of next
workflow = chain(
    fetch_data.s(url),
    process_data.s(),
    store_result.s()
)
workflow.delay()

# group: tasks run in parallel
parallel = group(
    send_email.s(user1),
    send_email.s(user2),
    send_email.s(user3)
)
parallel.delay()

# chord: parallel tasks → single callback when ALL complete
chord(
    group(process_chunk.s(chunk) for chunk in chunks)
)(aggregate_results.s())
```

#### Periodic Tasks with Celery Beat

Celery Beat is a scheduler that enqueues tasks on a schedule (like cron):

```python
# celeryconfig.py
from celery.schedules import crontab

beat_schedule = {
    'send-daily-digest': {
        'task': 'tasks.send_daily_digest',
        'schedule': crontab(hour=9, minute=0),  # every day at 9:00 AM
    },
    'cleanup-old-sessions': {
        'task': 'tasks.cleanup_sessions',
        'schedule': 3600.0,   # every hour (in seconds)
    },
}
```

```bash
# Run Beat scheduler alongside workers
celery -A tasks beat --loglevel=info
```

> ⚠️ **Only run ONE instance of Celery Beat** — running multiple Beat processes will trigger duplicate task scheduling!

#### Practical Platform Constraints

- **Replit / serverless platforms**: Running Celery + Redis + Workers requires persistent background processes, which many free-tier or serverless platforms don't support. Celery is designed for **traditional server deployments** (VPS, containers, on-prem).
- **Memory**: Workers are persistent processes — memory leaks accumulate. Use `worker_max_tasks_per_child` to periodically recycle workers.
- **Windows**: Celery's `prefork` pool doesn't work well on Windows (use `--pool=solo` for dev, or use Docker/WSL for production).
- **Debugging**: Tasks run in separate worker processes — `print()` statements appear in worker logs, not the web app logs. Use proper logging: `from celery.utils.log import get_task_logger`.

---

## Summary: Full Celery Stack Mental Model

```
User Action (HTTP Request)
        │
        ▼
   Flask / Django
   ┌─────────────────────────────┐
   │  validate input             │
   │  task.delay(args)  ─────────┼──► Redis/RabbitMQ [■■■■■■ Queue]
   │  return 202 Accepted        │                          │
   └─────────────────────────────┘                          │ (worker polls/receives)
                                                            ▼
                                                    Celery Worker
                                                    ┌────────────────────────┐
                                                    │  deserialize task       │
                                                    │  execute function       │
                                                    │  handle errors/retries  │
                                                    │  store result ──────────┼──► Redis/DB [Result Backend]
                                                    └────────────────────────┘

User polls /status/<task_id>
        │
        ▼
   Flask: AsyncResult(task_id).status / .result
        │
        ▼ reads from Result Backend
   {"status": "SUCCESS", "result": "..."}
        │
        ▼
   User sees completion ✅
```

**Key takeaway:** Celery lets you write background tasks as ordinary Python functions, handles all the complexity of message serialization, broker communication, worker coordination, failure handling, and result storage — so your application code stays clean and focused on business logic.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

