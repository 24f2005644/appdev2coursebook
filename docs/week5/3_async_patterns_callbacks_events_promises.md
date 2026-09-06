# 3. Asynchronous Patterns: Callbacks, Events & Promises



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **3. Asynchronous Patterns: Callbacks, Events & Promises**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 3.1 Callbacks & Higher-Order Functions

### The Problem With Synchronous / Blocking Functions

Consider the naive approach of treating an asynchronous operation as if it were synchronous:

```js
// ❌ Naive synchronous assumption — DOES NOT WORK for async
let result = doSomething();   // Assume doSomething() involves a network request
console.log(result);          // result is undefined — the data hasn't arrived yet!
```

Why does this fail?
- `doSomething()` starts the async operation and **returns immediately** (returns `undefined`)
- The network request is still in-flight in the background
- `console.log(result)` runs before the data ever arrives
- There is no built-in mechanism to "wait" for the result in synchronous code without blocking the thread

---

### The Callback Pattern: Continuation-Passing Style (CPS)

The solution is to **invert control** — instead of asking for a return value, you pass in a function (a **callback**) that the async operation should call when it finishes. This is known as **Continuation-Passing Style (CPS)**.

> Instead of: *"run this operation and give me the result"*
> We say: *"run this operation, and when you're done, call this function I'm giving you"*

```js
// ✅ Callback pattern (CPS)
function doSomething(callback) {
    // Simulate async work (e.g., network request)
    setTimeout(function() {
        const result = { data: 'response from server' };
        callback(result);    // Call the provided function when done
    }, 1000);
}

// We pass in a callback — control is handed to doSomething()
doSomething(function(result) {
    console.log(result.data);   // Runs 1 second later when data is ready
});

// This line runs immediately, without waiting
console.log('Request sent, waiting...');
```

**Output:**
```
Request sent, waiting...
response from server    (appears after ~1 second)
```

---

### Higher-Order Functions & Conditional Dispatch

A **Higher-Order Function** is a function that accepts other functions as arguments. In the async callback pattern, we commonly pass **two** callbacks — one for success and one for failure:

```js
function fetchData(url, successCB, failureCB) {
    // Simulate async operation that may succeed or fail
    setTimeout(function() {
        const networkError = Math.random() < 0.3;  // 30% chance of failure

        if (networkError) {
            failureCB(new Error('Network request failed'));   // Conditional dispatch
        } else {
            successCB({ status: 200, data: 'Hello from ' + url });
        }
    }, 1000);
}

// Calling the higher-order function with two callback arguments
fetchData(
    'https://api.example.com/data',

    function successCB(response) {
        console.log('✅ Success:', response.data);
    },

    function failureCB(error) {
        console.error('❌ Error:', error.message);
    }
);
```

The function **conditionally dispatches** to either `successCB` or `failureCB` depending on the outcome — it decides *which* callback to invoke, based on the result.

---

### The Pitfall: Callback Hell (Pyramid of Doom)

When you need to chain multiple async operations sequentially — each depending on the result of the previous — callbacks lead to deeply nested, hard-to-read code:

```js
// ❌ Callback Hell — deeply nested, hard to reason about
fetchUser(userId, function(user) {
    fetchOrders(user.id, function(orders) {
        fetchOrderDetails(orders[0].id, function(details) {
            fetchProductInfo(details.productId, function(product) {
                console.log('Product:', product.name);
                // Imagine error handling at EACH of these levels...
            }, function(err) { console.error(err); });
        }, function(err) { console.error(err); });
    }, function(err) { console.error(err); });
}, function(err) { console.error(err); });
```

Problems with this pattern:
- **Readability**: Logic flows horizontally and deeply rather than top-to-bottom
- **Error handling**: Each level needs its own error callback — no single place to catch all errors
- **Maintainability**: Adding a step means nesting another level
- **Inversion of Control**: We hand our callback to a function and **trust** it to call it correctly — but we have no guarantee it won't call it twice, never call it, or call it synchronously

---

## 3.2 Event-Driven Asynchrony

### The Event-Driven Model

A related but distinct asynchronous pattern is the **event-driven model**, where code reacts to events that occur at unpredictable times — user actions, network signals, timers, and system notifications.

The key distinction from plain callbacks:

| Pattern | When the callback is called |
|---|---|
| **Callback (CPS)** | Called once when a specific async operation completes |
| **Event Handler** | Called whenever a specific event occurs — could be zero times or many times |

---

### DOM Event Handlers: Functions Not Called Imperatively

In traditional imperative code, you call a function directly:
```js
greet('Vinay');   // You decide when and how to call it
```

With event handlers, you **register** a function with the browser's DOM and the **browser decides** when to call it — in response to user or system events:

```js
// You do NOT call this function directly — the browser calls it when the event fires
document.getElementById('submit-btn').addEventListener('click', function(event) {
    console.log('Button was clicked!');
    event.preventDefault();   // Prevent default form submission
});
```

The browser is now responsible for:
1. Monitoring for the event (click, keypress, scroll, resize, etc.)
2. Calling your registered function with an `event` object when it fires
3. Continuing to listen for future occurrences of the same event

---

### Registering Event Callbacks

You can register callbacks for any DOM event:

```js
// Click event on a button
document.querySelector('#my-btn').addEventListener('click', function(event) {
    console.log('Clicked at:', event.clientX, event.clientY);
});

// Keydown event on the document
document.addEventListener('keydown', function(event) {
    if (event.key === 'Enter') {
        console.log('Enter key pressed');
    }
});

// Form submission event
document.querySelector('#my-form').addEventListener('submit', function(event) {
    event.preventDefault();   // Stop browser from reloading the page
    const formData = new FormData(event.target);
    console.log('Form submitted:', Object.fromEntries(formData));
});

// Window resize event
window.addEventListener('resize', function() {
    console.log('Window size:', window.innerWidth, 'x', window.innerHeight);
});
```

The same callback mechanism works for custom events emitted by Vue components:
```js
// In Vue — listening to a custom event emitted by a child component
// Parent template:
// <ChildComponent @data-loaded="handleData" />
methods: {
    handleData(payload) {
        this.items = payload.items;
    }
}
```

---

## 3.3 Promises

### What is a Promise?

A **Promise** is a built-in JavaScript object that represents the **eventual completion or failure** of an asynchronous operation, and its resulting value.

It is called a "promise" because it is a commitment — *"I promise I will eventually give you a value or tell you I failed"*.

A Promise is always in one of three states:

```
┌─────────────┐
│   PENDING   │  ──► Operation is in progress, neither succeeded nor failed
└─────────────┘
       │
       ├── Operation succeeded ──►  ┌────────────┐
       │                            │  FULFILLED │  Has a resolved value
       │                            └────────────┘
       │
       └── Operation failed ──────► ┌────────────┐
                                    │  REJECTED  │  Has a reason (Error)
                                    └────────────┘
```

**Key rule**: Once a Promise transitions from `pending` to either `fulfilled` or `rejected`, it is **settled** and its state **never changes** again.

---

### The Promise Syntax: `.then(successCB, failureCB)`

Promises replace the callback pattern with a cleaner, chainable interface:

```js
// A function that returns a Promise
function doSomething() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            const success = true;

            if (success) {
                resolve({ data: 'server response' });   // Fulfill with a value
            } else {
                reject(new Error('Something went wrong'));  // Reject with a reason
            }
        }, 1000);
    });
}

// Consuming the Promise
doSomething()
    .then(
        function successCB(result) {
            console.log('✅ Fulfilled:', result.data);
        },
        function failureCB(error) {
            console.error('❌ Rejected:', error.message);
        }
    );
```

Or with the more common `.then().catch()` separation:

```js
doSomething()
    .then(function(result) {
        console.log('✅ Fulfilled:', result.data);
    })
    .catch(function(error) {
        console.error('❌ Rejected:', error.message);
    })
    .finally(function() {
        console.log('🏁 Done (always runs)');
    });
```

---

### More Than Syntax: Behavioral Guarantees

Promises are not just nicer syntax for callbacks. They come with **formal behavioral guarantees** that plain callbacks cannot provide:

| Guarantee | What It Means |
|---|---|
| **Settled only once** | A Promise can only be resolved or rejected one time — calling `resolve()` or `reject()` twice has no effect |
| **Run-to-completion** | `.then()` callbacks always run asynchronously (as microtasks) — never synchronously mid-stack, even if the Promise is already resolved |
| **Inversion of control recovered** | You own the `.then()` chain — the creator of the Promise cannot control which callbacks you attach or when |
| **Guaranteed callback invocation** | If you attach a `.then()`, it will be called exactly once — not zero times, not twice |
| **Error propagation** | A `throw` or `reject` inside any `.then()` automatically propagates down the chain to the nearest `.catch()` |

---

### Chaining and Composability

The most powerful aspect of Promises is **chaining** — each `.then()` returns a new Promise, allowing sequential async operations to be expressed as a flat, readable chain rather than nested callbacks:

```js
// ✅ Promise chaining — flat, readable, sequential async steps
fetchUser(userId)
    .then(function(user) {
        console.log('Got user:', user.name);
        return fetchOrders(user.id);          // Return a new Promise — chain continues
    })
    .then(function(orders) {
        console.log('Got orders:', orders.length);
        return fetchOrderDetails(orders[0].id);
    })
    .then(function(details) {
        console.log('Got details:', details);
        return fetchProductInfo(details.productId);
    })
    .then(function(product) {
        console.log('Product:', product.name);
    })
    .catch(function(error) {
        // ONE catch block handles errors from ANY step in the entire chain
        console.error('Something failed:', error.message);
    });
```

Compare this to the callback hell version from 3.1 — the same logic, but now:
- Reads top-to-bottom, like synchronous code
- One `.catch()` handles all errors from all steps
- Easy to add, remove, or reorder steps

---

### `async` / `await` — Syntactic Sugar Over Promises

`async`/`await` is a modern syntax introduced in ES2017 that makes Promise-based code look and behave almost identically to synchronous code:

```js
// Equivalent to the chain above, using async/await
async function loadProductForUser(userId) {
    try {
        const user    = await fetchUser(userId);           // Wait for Promise to settle
        const orders  = await fetchOrders(user.id);
        const details = await fetchOrderDetails(orders[0].id);
        const product = await fetchProductInfo(details.productId);
        console.log('Product:', product.name);
    } catch (error) {
        // Catches rejections from any awaited Promise above
        console.error('Something failed:', error.message);
    }
}
```

> **Important**: `async`/`await` does NOT introduce new async behavior — it is purely syntactic sugar over Promises. Under the hood, `await` pauses execution inside the `async` function and schedules the rest as a `.then()` microtask.

---

### Concrete Execution Trace: Sequential `await`

This is a critical example to understand — two Promises that are already running in the background, awaited one after the other:

```js
// Both Promises are created and start their timers IMMEDIATELY
const p1 = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('p1 resolved');   // Resolves after 5 seconds
    }, 5000);
});

const p2 = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('p2 resolved');   // Resolves after 3 seconds
    }, 3000);
});

async function fun() {
    console.log('start');          // 1. Runs immediately

    const b = await p1;           // 2. Pauses fun() here — waits for p1 (5s)
    console.log(b);               // 3. Prints after 5s: "p1 resolved"
    console.log('hello');         // 4. Prints immediately after step 3

    const a = await p2;           // 5. p2 already resolved at 3s — resumes instantly!
    console.log(a);               // 6. Prints immediately: "p2 resolved"
}

fun();
// Code outside fun() continues running here — fun() is non-blocking
console.log('fun() called, continuing outside...');
```

**Output (with timing):**
```
start                            ← t=0s  (synchronous, before first await)
fun() called, continuing...     ← t=0s  (outside code runs — fun() is non-blocking)
p1 resolved                      ← t=5s  (await p1 unblocks after 5 seconds)
hello                            ← t=5s  (immediately after p1 resolves)
p2 resolved                      ← t=5s  (p2 already resolved at t=3s — no extra wait!)
```

**The key insight — why total time is 5s, not 8s:**

```
Timeline:
  t=0s  ── [p1 timer starts] ─────────────────────────────────────► resolves at t=5s
  t=0s  ── [p2 timer starts] ──────────────────► resolves at t=3s

  fun() awaits p1 ──── waits until t=5s ────────────────────────────► unblocks
  fun() awaits p2 ──── p2 already done ────────────────────────────► unblocks immediately
```

Both timers start at `t=0` because both Promises are constructed before `await` is hit. `await` does **not** start or restart a Promise — it only waits for an already-running Promise to settle. So by the time `p1` resolves at `t=5s`, `p2` has already been resolved for 2 seconds — `await p2` returns instantly.

---

## 3.4 Promise Combinators: Running Multiple Promises Together

So far we've awaited Promises **sequentially** (one after another). But sometimes we need to run multiple async operations **concurrently** and coordinate their results. JavaScript provides four static `Promise` methods for this:

---

### `Promise.all()` — Wait for ALL to succeed

Takes an array of Promises. Resolves when **every** Promise in the array fulfills. Rejects **immediately** if **any one** Promise rejects (fail-fast).

```js
const p1 = new Promise((res) => setTimeout(() => res('p1 done'), 3000));
const p2 = new Promise((res) => setTimeout(() => res('p2 done'), 1000));
const p3 = new Promise((res) => setTimeout(() => res('p3 done'), 2000));

// All three run CONCURRENTLY — total wait = max(3s, 1s, 2s) = 3s
Promise.all([p1, p2, p3])
    .then(function(results) {
        console.log(results);
        // ['p1 done', 'p2 done', 'p3 done']  ← order matches input, not completion order
    })
    .catch(function(error) {
        // If ANY promise rejects, this runs immediately
        console.error('One failed:', error);
    });
```

**With async/await:**
```js
async function loadAll() {
    try {
        const [user, posts, comments] = await Promise.all([
            fetch('/api/user/1').then(r => r.json()),
            fetch('/api/posts').then(r => r.json()),
            fetch('/api/comments').then(r => r.json()),
        ]);
        console.log(user, posts, comments);  // All three loaded in parallel
    } catch (err) {
        console.error('At least one request failed:', err);
    }
}
```

> **Use case**: Fetching multiple independent resources simultaneously (e.g., user profile + dashboard stats + notifications). Much faster than awaiting them one by one.

> **Danger**: One rejection kills the entire batch. If you need results from the rest even when one fails, use `Promise.allSettled()`.

---

### `Promise.race()` — Settle with the FIRST to finish

Takes an array of Promises. Settles (resolves or rejects) as soon as the **first** Promise in the array settles — whichever comes first, wins. All other Promises are ignored (though they still run to completion in the background).

```js
const fast = new Promise((res) => setTimeout(() => res('fast (1s)'), 1000));
const slow = new Promise((res) => setTimeout(() => res('slow (4s)'), 4000));

Promise.race([fast, slow])
    .then(function(winner) {
        console.log('Winner:', winner);  // 'fast (1s)' — prints after 1 second
    });
```

**Use case — implementing a request timeout:**
```js
function withTimeout(promise, ms) {
    const timeout = new Promise((_, reject) =>
        setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
    );
    return Promise.race([promise, timeout]);
}

// If the API call takes longer than 3 seconds, reject with a timeout error
withTimeout(fetch('/api/slow-endpoint'), 3000)
    .then(r => r.json())
    .then(data => console.log('Got data:', data))
    .catch(err => console.error(err.message));  // 'Timed out after 3000ms'
```

> **Use case**: Timeout patterns, taking the fastest of multiple redundant server calls (e.g., querying two CDN nodes and using whichever responds first).

---

### `Promise.allSettled()` — Wait for ALL, regardless of outcome

Takes an array of Promises. **Always resolves** (never rejects), once every Promise has settled — regardless of whether each one fulfilled or rejected. Returns an array of result objects describing each outcome.

```js
const p1 = new Promise((res) => setTimeout(() => res('success A'), 1000));
const p2 = new Promise((_, rej) => setTimeout(() => rej(new Error('failed B')), 2000));
const p3 = new Promise((res) => setTimeout(() => res('success C'), 1500));

Promise.allSettled([p1, p2, p3])
    .then(function(results) {
        results.forEach(result => {
            if (result.status === 'fulfilled') {
                console.log('✅ Fulfilled:', result.value);
            } else {
                console.log('❌ Rejected:', result.reason.message);
            }
        });
    });

// Output (after ~2 seconds):
// ✅ Fulfilled: success A
// ❌ Rejected: failed B
// ✅ Fulfilled: success C
```

Each result object has the shape:
```js
// On success:
{ status: 'fulfilled', value: <resolved value> }

// On failure:
{ status: 'rejected', reason: <Error object> }
```

> **Use case**: When you want to attempt multiple operations and process all results, even if some fail. For example: sending notifications to multiple users — you don't want one failure to hide all the successes.

---

### Combinator Comparison Table

| Method | Resolves When | Rejects When | Returns | Best For |
|---|---|---|---|---|
| `Promise.all()` | **ALL** fulfill | **ANY ONE** rejects (fail-fast) | Array of resolved values (in input order) | Parallel independent requests, all must succeed |
| `Promise.race()` | **FIRST** to settle (fulfill or reject) | First settles as rejection | Single value/error from the winner | Timeout patterns, fastest-wins scenarios |
| `Promise.allSettled()` | **ALL** settle (any outcome) | **Never** rejects | Array of `{status, value/reason}` objects | Batch operations where partial failure is acceptable |
| `Promise.any()` | **FIRST** to **fulfill** | ALL reject | Single resolved value from the winner | Try multiple sources, use first success |

---

## Summary

```
Topic 3: Asynchronous Patterns — Callbacks, Events & Promises
│
├── 3.1 Callbacks & Higher-Order Functions
│     ├── Problem    ──► `let result = doSomething()` returns undefined for async ops
│     ├── CPS        ──► Pass a callback function; let the async operation call it when done
│     ├── HOF        ──► Conditionally dispatch successCB or failureCB based on outcome
│     └── Pitfall    ──► Callback Hell — deeply nested, hard to read, no central error handling
│
├── 3.2 Event-Driven Asynchrony
│     ├── Model      ──► Register a function; the browser/DOM calls it when an event fires
│     ├── Key trait  ──► Not called imperatively — called by the system at unpredictable times
│     └── Scope      ──► User events (click, keydown, submit, resize) + custom Vue events
│
├── 3.3 Promises
│     ├── States     ──► Pending → Fulfilled (resolved value) or Rejected (error reason)
│     ├── Syntax     ──► .then(successCB, failureCB) / .catch() / .finally()
│     ├── Guarantees ──► Settled once, run-to-completion, controlled inversion, error propagation
│     ├── Chaining   ──► Flat sequential async steps, single .catch() for entire chain
│     ├── async/await ──► Syntactic sugar; await pauses function, not thread
│     └── Sequential await insight ──► Both Promises run concurrently; await only WAITS, doesn't START
│
└── 3.4 Promise Combinators
      ├── Promise.all()        ──► All must fulfill; one rejection = immediate fail
      ├── Promise.race()       ──► First to settle wins; useful for timeout patterns
      ├── Promise.allSettled() ──► Always resolves; reports each outcome individually
      └── Promise.any()        ──► First to FULFILL wins; ignores rejections unless all fail
```

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
