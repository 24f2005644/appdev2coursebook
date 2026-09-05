# 2. Asynchronous JavaScript Fundamentals

---

## 2.1 The Need for Asynchronous Operations

### The Core Problem: Factors Outside Our Control

When a web app fetches data from a server, the duration of that operation is determined by factors **completely outside the control** of both the client and the server code:

- **Network latency** — physical distance data must travel, routing hops, undersea cables
- **Network disruptions** — packet loss, mobile handoffs, intermittent connectivity
- **Server load** — the remote server may be handling thousands of simultaneous requests, causing queuing delays
- **DNS resolution delays** — converting a domain name to an IP address takes time
- **TLS/SSL handshake** — establishing a secure encrypted connection adds round-trips

There is **no way to predict or guarantee** how long any given HTTP request will take — it could be 50ms or 5 seconds.

---

### The Single-Threaded Constraint

JavaScript in the browser runs on a **single main thread**. This one thread is responsible for:
- Executing all JavaScript code
- Responding to user interactions (clicks, keyboard, scroll)
- Running CSS layout calculations
- Painting frames to the screen (ideally 60 fps = 1 frame every ~16ms)

There is no second thread to offload work to (within the main JS runtime itself).

#### What Happens If We Block This Thread?

Imagine JavaScript had a hypothetical `fetchSync()` that waited for the response before returning:

```js
// ⚠️ Hypothetical blocking example — this is NOT how real fetch() works
const data = fetchSync('https://api.example.com/data');  // Takes 2 seconds
console.log(data);  // Only runs after the 2-second wait
```

During those 2 seconds:
- The single thread is **stuck** waiting
- The browser **cannot process click events** — buttons don't respond
- The browser **cannot scroll** the page
- The browser **cannot repaint** — animations freeze
- The user sees a completely unresponsive, frozen page
- In severe cases, the browser shows a **"Page Unresponsive"** dialog

This is completely unacceptable for user experience.

---

### The Solution: The Non-Blocking Execution Model

Instead of waiting, JavaScript uses a **non-blocking** approach:

1. **Start the operation** — hand the request off to the browser's background APIs
2. **Return immediately** — the main thread is freed to keep running other code
3. **Register a callback** — a function to call when the operation eventually completes
4. **Continue executing** — the main thread processes other events, renders frames, handles clicks
5. **Handle the result** — when the data arrives, the registered callback is executed to update the UI

```
❌ Blocking (Synchronous):
  Main Thread: [Start fetch]────────────────────────[Wait 2s]────────────────[Process data]
  User Input:  ─────────────────────────[FROZEN - clicks ignored]────────────►

✅ Non-Blocking (Asynchronous):
  Main Thread: [Start fetch][Other code][Handle clicks][Render UI]...[Process data when ready]
  Background:              [──────────── Network I/O (2s) ───────────► callback queued]
```

---

## 2.2 JavaScript Execution Model: Call Stack & Event Loop

JavaScript's ability to handle async operations on a single thread is made possible by a carefully designed runtime architecture. There are four key components:

---

### Component 1: The Call Stack

The **Call Stack** is a **Last-In, First-Out (LIFO)** data structure that tracks what code is currently being executed.

- When a function is **called**, it is **pushed** onto the stack
- When a function **returns**, it is **popped** off the stack
- JavaScript always executes whatever is **at the top** of the stack

```js
function greet(name) {
    return 'Hello, ' + name;
}

function main() {
    const msg = greet('Vinay');  // greet() is pushed on top of main()
    console.log(msg);
}

main();
```

```
Call Stack during execution:

Step 1:   [main()]                  ← main() is called, pushed
Step 2:   [greet()]                 ← greet() is called from inside main(), pushed on top
          [main()]
Step 3:   [main()]                  ← greet() returns, popped off
Step 4:   [console.log()]           ← log() is called, pushed
          [main()]
Step 5:   [main()]                  ← log() returns, popped
Step 6:   (empty)                   ← main() returns, popped — stack is now empty
```

**Key rule**: JavaScript only executes one thing at a time. The Call Stack can only hold one active execution context at the top.

---

### Component 2: Web APIs (Browser-Provided Background Threads)

The browser provides a set of **Web APIs** — features implemented outside the JavaScript engine itself, in the browser's C++ internals — that can execute concurrently in the background:

| Web API | What It Does |
|---|---|
| `fetch()` / `XMLHttpRequest` | Makes HTTP network requests using the browser's network layer |
| `setTimeout()` / `setInterval()` | Runs timers using a dedicated timer thread |
| DOM event listeners | Listens for user input events (clicks, keypresses, scroll) |
| `navigator.geolocation` | Handles GPS/location lookups |
| Web Workers | Dedicated parallel worker threads for CPU-intensive tasks |

When you call `fetch()`, JavaScript **registers** the request with the browser's network API and immediately returns — the actual network I/O happens in background threads outside the JS engine.

---

### Component 3: The Callback Queue (Task Queue)

When a Web API's background operation completes (e.g., the HTTP response arrives), it does **not** immediately run your callback. Instead, it places your callback function into the **Callback Queue** (also called the Task Queue or Macrotask Queue).

The callback waits here until the Call Stack is completely empty.

---

### Component 4: The Event Loop

The **Event Loop** is a continuously running mechanism that coordinates everything:

```
while (true) {
    if (callStack.isEmpty()) {
        if (microtaskQueue.hasItems()) {
            callStack.push(microtaskQueue.dequeue());  // Drain ALL microtasks first
        } else if (callbackQueue.hasItems()) {
            callStack.push(callbackQueue.dequeue());   // Then pick ONE macrotask
        }
    }
    // (Also checks if a browser repaint frame is needed between macrotasks)
}
```

In plain English:
1. **Watch the Call Stack** — if it's not empty, do nothing and let execution continue
2. **Once the Call Stack empties** — check the Microtask Queue and drain it completely
3. **Then** — pick the oldest callback from the Callback Queue, push it onto the stack, and run it
4. **Repeat** forever

---

### Putting It All Together: A Full Trace

```js
console.log('A');                          // 1

setTimeout(function() {
    console.log('B');                      // 4 (runs last)
}, 0);

Promise.resolve().then(function() {
    console.log('C');                      // 3 (microtask, before setTimeout)
});

console.log('D');                          // 2
```

**Output:**
```
A
D
C
B
```

**Why?**
1. `console.log('A')` → runs immediately (synchronous)
2. `setTimeout(...)` → handed to Web API; callback queued in **Macrotask Queue** after 0ms
3. `Promise.resolve().then(...)` → `.then()` callback queued in **Microtask Queue**
4. `console.log('D')` → runs immediately (synchronous)
5. Call Stack is now **empty** → Event Loop kicks in
6. **Microtask Queue** is drained first → `console.log('C')` runs
7. **Macrotask Queue** is checked next → `console.log('B')` runs

> **Key insight**: Even a `setTimeout` with `0ms` delay **cannot** run before a queued Promise `.then()`, because microtasks are always drained before macrotasks.

---

### External Specifications
- **WHATWG HTML Living Standard — Event Loops**: The official browser specification defining exact event loop behavior
- **MDN Web Docs — The Event Loop**: `developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop`

---

## 2.3 Concurrency vs. Parallelism

These two terms are frequently confused but describe fundamentally different concepts.

---

### Definitions

**Concurrency** — *dealing with multiple things at once*
> Multiple tasks are **in progress** simultaneously, but they may not be executing at the exact same physical instant. They make progress by taking turns on a shared resource (time-multiplexing / interleaving).

**Parallelism** — *doing multiple things at once*
> Multiple tasks are **physically executing** at the exact same instant on separate processing units (different CPU cores or processors).

---

### Visual Analogy

```
CONCURRENCY — a single barista making multiple coffees:
  [Coffee A: grind beans] → [Coffee B: pour water] → [Coffee A: froth milk] → [Coffee B: finish]
  One person, multiple tasks interleaved over time.

PARALLELISM — multiple baristas each making their own coffee:
  Barista 1: [═══════════ Coffee A ═══════════]
  Barista 2: [═══════════ Coffee B ═══════════]
  Multiple people, each task running simultaneously.
```

---

### The Fundamental Relationship

> **Parallelism implies concurrency, but concurrency does NOT require parallelism.**

- If tasks are running in parallel (on different cores), they are by definition also concurrent
- But tasks can be concurrent (interleaved, time-multiplexed) on a single core without any parallelism

---

### How JavaScript Implements Concurrency

JavaScript is **single-threaded** — it achieves concurrency through **interleaving** via the Event Loop, not through parallelism:

```
JS Main Thread (single core):
  [Code A] [Start fetch] [Code B] [Handle click] [Code C] ... [HTTP response callback]
              ↓
  Browser Network Thread (parallel, C++):
  [──────────────── actual HTTP I/O ────────────────► queues callback when done]
```

The JavaScript code itself runs one piece at a time (no parallelism within JS code), but the **browser as a whole is multithreaded** — the network thread, timer thread, and rendering thread all run in parallel with the JS thread. Only the callbacks registered with those APIs get queued back onto the single JS thread.

---

### Concurrency in JavaScript Runtimes

| Environment | Mechanism | Concurrency or Parallelism? |
|---|---|---|
| **Single JS main thread** | Event Loop + Callback Queue | **Concurrency** (interleaved on one thread) |
| **`setTimeout` / `setInterval`** | Browser timer thread queues callbacks | **Concurrency** (callback runs on main thread when due) |
| **`fetch()` / network requests** | Browser network thread; callback queued on completion | **Concurrency** (callback runs on main thread when response arrives) |
| **Web Workers** | Separate JS execution thread per worker | **Parallelism** (truly simultaneous JS execution on different CPU cores) |

```js
// Web Workers: genuine parallelism in the browser
// main.js (runs on main thread)
const worker = new Worker('worker.js');    // spawns a new parallel thread

worker.postMessage({ task: 'heavyCalc', data: [1,2,3,4,5] });

worker.onmessage = function(event) {
    // Runs on main thread after worker finishes
    console.log('Result:', event.data.result);
};

// worker.js (runs on a SEPARATE parallel thread — does not block the UI)
self.onmessage = function(event) {
    const result = event.data.data.reduce((a, b) => a + b, 0);
    self.postMessage({ result });
};
```

---

## Summary

```
Topic 2: Asynchronous JavaScript Fundamentals
│
├── 2.1 Why Async Is Needed
│     ├── Unpredictable factors  ──► latency, disruptions, server load
│     ├── Single-threaded JS     ──► blocking = frozen UI, unresponsive page
│     └── Non-blocking model     ──► hand off I/O, continue running, handle callback later
│
├── 2.2 Execution Model
│     ├── Call Stack       ──► LIFO, one execution at a time, current synchronous code
│     ├── Web APIs         ──► Browser background threads (network, timers, DOM events)
│     ├── Callback Queue   ──► Completed callbacks wait here (Macrotask Queue)
│     └── Event Loop       ──► Drains Microtasks → picks Macrotask → repeat, when stack is empty
│
└── 2.3 Concurrency vs. Parallelism
      ├── Concurrency  ──► multiple tasks in progress, interleaved (can be single-core)
      ├── Parallelism  ──► multiple tasks executing simultaneously (requires multiple cores)
      ├── Law          ──► Parallelism ⊂ Concurrency (not the other way around)
      └── JS Reality   ──► Event loop = concurrent; Web Workers = parallel
```
