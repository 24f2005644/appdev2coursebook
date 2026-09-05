# 3. Asynchrony & The JavaScript Runtime

---

## 3.1 Asynchrony Overview

### The Core Problem
Most interesting programs need to do things that take **unpredictable amounts of time**:
- Fetch data from a server over a network
- Read a file from disk
- Wait for a user to click a button
- Wait for a timer to expire

In a **synchronous** model, the program would simply *wait* (block) until the operation completes before doing anything else. This is catastrophic for user-facing applications — the entire page would freeze while waiting for a network response.

> **Asynchrony** is the ability to *initiate* a long-running operation and then *continue doing other work* until that operation signals completion — rather than waiting idle.

---

### Synchronous vs Asynchronous

```
Synchronous:
  [Start Task A] → [Wait...Wait...Wait...] → [Task A Done] → [Start Task B]

Asynchronous:
  [Start Task A] → [Continue with Task B, Task C...] → [Task A signals done] → [Handle result]
```

JavaScript handles asynchrony through three mechanisms (in order of modernity):
1. **Callbacks** — Pass a function to be called when work is done
2. **Promises** — A placeholder object representing a future value
3. **`async` / `await`** — Syntactic sugar over Promises for cleaner code

All three are built on the same underlying machinery: the **Call Stack**, **Task Queue**, and **Event Loop**.

---

## 3.2 Execution Context & The Call Stack

### What is an Execution Context?
Every time a function is called, JavaScript creates an **execution context** — a record containing:
- The function's local variables
- The value of `this`
- A reference back to the calling context (where to return to when done)

Think of it as a **saved snapshot** of the function's state while it runs.

---

### The Call Stack
> The **call stack** is a LIFO (Last-In, First-Out) data structure that tracks which functions are currently executing.

When a function is **called**, its execution context is **pushed** onto the stack.
When a function **returns**, its context is **popped** off the stack.
Execution resumes in the context below it.

---

### Step-by-Step Walkthrough

```javascript
function h() {
  return 1 + 1;
}

function g() {
  return h();
}

function f() {
  return g();
}

f();  // start here
```

| Step | Event | Stack (top → bottom) |
|------|-------|----------------------|
| 1 | `f()` is called | `f` |
| 2 | `f` calls `g()` | `g` → `f` |
| 3 | `g` calls `h()` | `h` → `g` → `f` |
| 4 | `h` returns `2` | `g` → `f` ← `h` popped |
| 5 | `g` returns `2` | `f` ← `g` popped |
| 6 | `f` returns `2` | *(empty)* ← `f` popped |

At step 3, `h` is on top — it executes next. The stack **preserves the return path** back through `g` → `f` → `main`.

---

### Stack Overflow
The call stack has a finite size. Infinitely recursive functions overflow it:

```javascript
function infinite() {
  return infinite();  // calls itself forever
}
infinite();  // ❌ RangeError: Maximum call stack size exceeded
```

---

### Visualization
The tool **Loupe** (by Philip Roberts) is a widely used visual simulator for the call stack, event loop, and task queue. Paste code at [latentflip.com/loupe](http://latentflip.com/loupe) to watch the stack animate in real time.

---

## 3.3 Event Loop and Task Queue

### The Challenge
The call stack can only do one thing at a time. But browsers need to handle many concurrent things: network responses, user clicks, timers, DOM events. How?

The answer: **the JavaScript runtime is more than just the engine**. It includes:

```
┌──────────────────────────────────────────────────────────────┐
│                    JavaScript Runtime                        │
│                                                              │
│  ┌─────────────┐    ┌──────────────────┐   ┌─────────────┐  │
│  │  Call Stack │    │   Web APIs /     │   │ Task Queue  │  │
│  │             │    │   Node APIs      │   │ (Callback   │  │
│  │  main()     │    │                  │   │  Queue)     │  │
│  │  f()        │    │  setTimeout      │   │             │  │
│  │  g()        │    │  fetch (XHR)     │   │ callback1   │  │
│  │             │    │  DOM events      │   │ callback2   │  │
│  └──────┬──────┘    └────────┬─────────┘   └──────┬──────┘  │
│         │                   │                     │          │
│         └──────── Event Loop monitors both ───────┘          │
└──────────────────────────────────────────────────────────────┘
```

---

### The Task Queue (Callback Queue)
> The **Task Queue** is a FIFO (First-In, First-Out) queue that holds **callbacks waiting to be executed** — callbacks from completed I/O, timers, click events, network responses, etc.

When you call `setTimeout(fn, 1000)`:
1. The timer is handed off to the **Web API** (outside the JS engine)
2. The JS engine continues executing (non-blocking!)
3. After 1000ms, the Web API **enqueues** `fn` onto the Task Queue
4. The Event Loop picks it up when the stack is empty

---

### The Event Loop
> The **Event Loop** is a continuous loop that monitors the call stack and the task queue. When the stack is **empty**, it dequeues the next task from the queue and pushes it onto the stack for execution.

```
while (true) {
  if (callStack.isEmpty() && taskQueue.isNotEmpty()) {
    callStack.push(taskQueue.dequeue());
  }
}
```

This is conceptually what the event loop does — it's the bridge between the Web APIs and your JavaScript code.

---

### Illustrated Example: `setTimeout`

```javascript
console.log("1 - Start");

setTimeout(() => {
  console.log("3 - Timeout callback");
}, 0);   // ← 0ms delay!

console.log("2 - End");
```

**Output:**
```
1 - Start
2 - End
3 - Timeout callback
```

> Even with a **0ms** delay, `"3"` prints last! Because:
> 1. `"1 - Start"` executes on the stack
> 2. `setTimeout` hands callback to Web API (even for 0ms, it goes through the queue)
> 3. `"2 - End"` executes on the stack
> 4. Stack is now empty → Event Loop dequeues the callback → `"3"` executes

This is the most commonly asked interview question about the event loop.

---

### Run-to-Completion Guarantee
> Once a function starts executing on the call stack, it **runs to completion** — the event loop will **never** preempt it midway to run something else.

```javascript
setTimeout(() => console.log("callback"), 0);

// This long loop will run entirely before the callback runs
for (let i = 0; i < 1_000_000_000; i++) { /* blocking! */ }

console.log("loop done");
// Output: "loop done", THEN "callback"
```

The callback had to wait the entire billion iterations. The event loop only checks the queue **between** tasks, never during one.

---

## 3.4 The Single-Threaded Nature & Blocking the Browser

### JavaScript is Single-Threaded
> JavaScript runs on a **single thread** — there is only one call stack, and only one piece of code executes at any given instant.

This is a deliberate design choice (simplicity, no race conditions in shared DOM access), but it has consequences.

---

### What "Blocking" Means
**Blocking** means occupying the single thread for an extended time without yielding, which prevents:
- UI repaints (page appears frozen)
- Processing user interactions (clicks, typing)
- Running any other JavaScript (including event handlers)

**Classic blocking scenario — synchronous network request:**
```javascript
// NEVER do this in a browser
const xhr = new XMLHttpRequest();
xhr.open('GET', 'https://api.example.com/data', false);  // false = synchronous!
xhr.send();  // browser FREEZES here until response arrives
console.log(xhr.responseText);
```

**CPU-intensive blocking:**
```javascript
// This freezes the browser for the duration of the computation
function expensiveComputation() {
  let result = 0;
  for (let i = 0; i < 10_000_000_000; i++) {
    result += Math.sqrt(i);
  }
  return result;
}
expensiveComputation();  // browser is unresponsive during this
```

---

### Mitigation Strategies

| Strategy | How | Use Case |
|----------|-----|----------|
| **Async APIs** | Use asynchronous versions of all I/O | Network, file, timers |
| **`setTimeout(fn, 0)`** | Defer heavy work to future task queue slot | Break up long computations |
| **Web Workers** | True multi-threading for CPU-heavy tasks | Image processing, cryptography |
| **`requestAnimationFrame`** | Schedule work aligned with browser paint cycle | Animations |

> The golden rule: **keep the call stack clear**. Every millisecond you block the thread, you're degrading the user experience.

---

## 3.5 Callbacks in Practice

### The Rationale
Callbacks are the original solution to asynchrony:

> Instead of *waiting* for a long operation to complete, you hand the runtime a **callback function** — "when this is done, call this function with the result".

The main thread is freed up to do other work while the operation runs in the background (handled by Web/Node APIs), and the callback is called only when needed.

---

### Node.js File I/O — The Canonical Example

#### Synchronous (Blocking) — `fs.readFileSync`
```javascript
const fs = require('fs');

try {
  const data = fs.readFileSync('/path/to/file.txt', 'utf8');
  console.log(data);       // file content
  console.log("Done");     // runs AFTER file is fully read
} catch (err) {
  console.error("Error:", err.message);
}
```

| Property | Value |
|----------|-------|
| **Execution** | Blocks the thread until file is fully read |
| **Error handling** | `try...catch` |
| **Use case** | Startup config files, CLI scripts where blocking is acceptable |

---

#### Asynchronous (Non-Blocking) — `fs.readFile`
```javascript
const fs = require('fs');

console.log("Before read");

fs.readFile('/path/to/file.txt', 'utf8', (err, data) => {
  // This callback runs when the file read is complete
  if (err) {
    console.error("Error:", err.message);
    return;
  }
  console.log(data);  // file content
});

console.log("After read");  // runs IMMEDIATELY, doesn't wait
```

**Output:**
```
Before read
After read
[file contents]   ← arrives later, when I/O completes
```

| Property | Value |
|----------|-------|
| **Execution** | Non-blocking — continues immediately |
| **Error handling** | **Error-first callback**: `(err, data)` pattern |
| **Use case** | Any I/O in production servers |

---

### The Error-First Callback Convention
Node.js established a universal callback signature:

```javascript
function callback(err, result) {
  if (err) {
    // something went wrong — handle the error
    return;
  }
  // success — use result
}
```

- **First argument** is always the error (`null` if no error)
- **Second argument** (and beyond) is the result data
- Always check `err` first before using `result`

This became the universal Node.js convention, used by virtually every core API and thousands of npm packages.

---

### Callback Hell
When operations depend on each other, callbacks nest deeply — this is known as **callback hell** or the **pyramid of doom**:

```javascript
fs.readFile('file1.txt', 'utf8', (err, data1) => {
  if (err) return handleError(err);
  
  fs.readFile('file2.txt', 'utf8', (err, data2) => {
    if (err) return handleError(err);
    
    fs.writeFile('output.txt', data1 + data2, (err) => {
      if (err) return handleError(err);
      
      db.save(outputData, (err, result) => {
        if (err) return handleError(err);
        
        // deeper and deeper...
      });
    });
  });
});
```

Problems:
- Hard to read and reason about
- Error handling duplicated at every level
- Difficult to handle errors in a centralized way
- Complex control flow (parallel operations, conditionals) becomes unwieldy

This is what motivated the design of **Promises**.

---

## 3.6 Modern Asynchronous Patterns

### Single-Threaded Throughput
Counterintuitively, JavaScript's single-threaded + async model can achieve **high throughput** for I/O-heavy workloads.

**Why?** Because most server time is spent *waiting* — for database queries, file reads, network responses. With async I/O, the thread serves many requests concurrently by processing them in between waits, rather than dedicating a thread per request (like Java servers traditionally did).

> This is the key insight behind Node.js's performance: **event-driven, non-blocking I/O** on a single thread outperforms thread-per-request models for I/O-heavy workloads.

---

### Promises — The Next Evolution

> A **Promise** is an object representing the **eventual completion (or failure)** of an asynchronous operation and its resulting value.

A Promise is in one of three states:

```
Pending  →  Fulfilled (resolved with a value)
         →  Rejected  (failed with a reason/error)
```

```javascript
const fs = require('fs/promises');  // Promise-based fs API

fs.readFile('/path/to/file.txt', 'utf8')
  .then(data => {
    console.log(data);           // runs on success
    return processData(data);    // can chain further .then()
  })
  .then(result => {
    console.log("Processed:", result);
  })
  .catch(err => {
    console.error("Error:", err.message);  // catches ANY error in the chain
  })
  .finally(() => {
    console.log("Done, whether success or failure");
  });
```

**Advantages over callbacks:**
- **Chainable** — `.then()` returns a new Promise, enabling clean sequential steps
- **Centralized error handling** — one `.catch()` handles errors from any step in the chain
- **Composable** — `Promise.all()`, `Promise.race()`, `Promise.allSettled()` for parallel operations

**Promise.all() — Run in Parallel:**
```javascript
const [file1, file2] = await Promise.all([
  fs.readFile('file1.txt', 'utf8'),
  fs.readFile('file2.txt', 'utf8')
]);
// Both reads happen concurrently — much faster than sequential
```

---

### `async` / `await` — Synchronous-Looking Async Code

> `async`/`await` is syntactic sugar over Promises that lets you write asynchronous code that **reads like synchronous code**, without changing the underlying non-blocking behavior.

```javascript
const fs = require('fs/promises');

async function processFiles() {
  try {
    const data1 = await fs.readFile('file1.txt', 'utf8');  // waits, but doesn't block
    const data2 = await fs.readFile('file2.txt', 'utf8');
    
    await fs.writeFile('output.txt', data1 + data2);
    console.log("Done!");
  } catch (err) {
    console.error("Error:", err.message);   // one try/catch for all errors
  }
}

processFiles();
```

Compare this to the callback hell version — same logic, dramatically cleaner.

**Rules:**
- `async` before a function declaration makes it return a Promise automatically
- `await` can only be used **inside** an `async` function
- `await` pauses execution of the *current async function* until the Promise resolves, but **does not block the thread** — other tasks can run while waiting

---

### The Full Evolution at a Glance

```
Callbacks (1990s–2010s)
  ↓  Hard to compose, callback hell
Promises (ES6, 2015)
  ↓  Better chaining, centralized catch, still .then() chains
async/await (ES8, 2017)
  → Cleanest syntax, synchronous-looking, still non-blocking
```

All three are still in use today. `async`/`await` is the modern default, but understanding callbacks and Promises is essential for reading existing code and understanding what `async`/`await` compiles to.

---

## Summary

| Concept | Key Idea | Key Term |
|---------|----------|----------|
| **Call Stack** | LIFO tracker of function execution contexts | Push / Pop |
| **Run-to-Completion** | No preemption mid-task; a task runs fully before the next | Guarantee |
| **Web / Node APIs** | Handle I/O outside the JS engine (timers, network, file) | Background tasks |
| **Task Queue** | FIFO queue of callbacks waiting to execute | Callback Queue |
| **Event Loop** | Moves tasks from queue → stack when stack is empty | The bridge |
| **Single Thread** | Only one piece of code runs at a time | One call stack |
| **Blocking** | Occupying the thread; prevents UI repaints & events | CPU-bound code |
| **Callbacks** | Function passed to run when async work completes | Error-first `(err, data)` |
| **Callback Hell** | Deeply nested callbacks, hard to manage | Pyramid of doom |
| **Promises** | Object representing a future value; chainable | `.then()`, `.catch()` |
| **`async`/`await`** | Synchronous-looking syntax over Promises | `await`, `async function` |
