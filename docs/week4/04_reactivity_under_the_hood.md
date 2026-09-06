# 4. Reactivity Under the Hood



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **4. Reactivity Under the Hood**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

> **Module**: MAD II — Week 4 | **Topic**: Vue.js Fundamentals
> **Subtopics**: 4.1 → 4.4

---

## 4.1 How Does Reactivity Work? (Tracking Access & Mutations)

### The Core Question

We've been using Vue's reactivity system all along — change `data`, template updates. But *how* does Vue know something changed? JavaScript doesn't have a built-in "notify me when this variable changes" mechanism.

Consider this plain JavaScript:

```javascript
let count = 0;

// There's no native way to say:
// "Hey, whenever 'count' changes, run this function"

count = 5;  // JavaScript doesn't fire any event here
```

Yet in Vue, this just works:

```javascript
data() {
  return { count: 0 }
},
methods: {
  increment() {
    this.count++;   // ← Template magically re-renders. How?
  }
}
```

The answer lies in **property interception**.

---

### The Two Operations Vue Must Detect

For the reactivity system to work, Vue needs to intercept exactly **two primitive operations** on every piece of reactive state:

```
┌─────────────────────────────────────────────────────────────┐
│                   TWO CRITICAL OPERATIONS                   │
│                                                             │
│  1. READ / GET   →  When is this data being accessed?       │
│                     (used to build the dependency graph)    │
│                                                             │
│  2. WRITE / SET  →  When is this data being mutated?        │
│                     (used to trigger re-renders)            │
└─────────────────────────────────────────────────────────────┘
```

**Why detect reads (GET)?**

When a component renders, it reads reactive data. Vue watches *which* data is read during that render. This builds a **dependency map**: "Component A depends on `count` and `username`". Now Vue knows: if `count` changes → re-render Component A. If some *other* property changes that Component A never reads → don't bother re-rendering it.

**Why detect writes (SET)?**

When data is mutated, Vue uses the dependency map built during reads to know *who* needs to be notified. It triggers all the watchers and component re-renders that were registered as dependents during the GET phase.

```
Component renders
      │
      ▼ (reads data.count, data.username during render)
Vue tracks: "Component A depends on → count, username"
      │
      ▼
Later: data.count = 5  (a SET)
      │
Vue consults dependency map → "Who depends on count? → Component A"
      │
      ▼
Schedules Component A for re-render → DOM updates
```

This is the **Dependency Tracking System** — the heart of Vue's reactivity.

---

### The Fundamental Challenge

JavaScript objects are dynamic — you can add, remove, or change properties at any time. There's no native way to observe this. So Vue had to find a way to **wrap** JavaScript property access and mutation with its own interceptor logic.

The solution in **Vue 2**: `Object.defineProperty()` — an ES5 API.
The solution in **Vue 3**: JavaScript `Proxy` objects (ES6+) — more powerful and capable.

We'll study the `Object.defineProperty()` approach (Vue 2 / conceptual basis) in detail.

---

## 4.2 JavaScript Property Interception: `Object.defineProperty()`

### What is `Object.defineProperty()`?

`Object.defineProperty()` is a **native JavaScript API** (available since ES5 / 2009) that allows you to define or modify a property on an object with **fine-grained control** over its behavior — including attaching custom **getter** and **setter** functions.

**Standard property** (the way you normally use objects):
```javascript
const obj = { name: "Vue" };
// obj.name is a simple data property
// Accessing obj.name just returns "Vue"
// Setting obj.name = "React" just updates the stored value
```

**Property with `Object.defineProperty()`**:
```javascript
const obj = {};

Object.defineProperty(obj, 'name', {
  // These are called "property descriptors"
  enumerable: true,       // Appears in for...in loops, Object.keys()
  configurable: true,     // Can be redefined later
  get() {
    // Custom logic runs when someone reads obj.name
    return "Vue";
  },
  set(newValue) {
    // Custom logic runs when someone writes obj.name = "..."
    console.log(`Setting name to: ${newValue}`);
  }
});
```

**Full signature**:

```javascript
Object.defineProperty(targetObject, propertyName, descriptorObject)
```

| Parameter | Description |
|---|---|
| `targetObject` | The object to define the property on |
| `propertyName` | The name of the property (as a string) |
| `descriptorObject` | Configuration: `get`, `set`, `value`, `writable`, `enumerable`, `configurable` |

---

### Data Descriptors vs Accessor Descriptors

| Type | Keys Used | Description |
|---|---|---|
| **Data descriptor** | `value`, `writable` | Stores a static value; `writable` controls if it can be changed |
| **Accessor descriptor** | `get`, `set` | No stored value; uses functions to compute/intercept access |

> **Vue's reactivity uses Accessor Descriptors** — `get` + `set` — not data descriptors.

A property **cannot be both** a data descriptor and an accessor descriptor simultaneously.

---

### Why This Is the Perfect Hook for Reactivity

`Object.defineProperty()` is the perfect tool because:
- It's **transparent** to the user: `obj.count` and `obj.count = 5` look like normal property access, but secretly run your custom functions.
- It's **part of the JavaScript spec** — no browser extensions, no monkey-patching, no overhead of a special API.
- It can be applied **after** an object is created, making it possible to take a plain `data()` object and reactify it.

---

## 4.3 Implementation Step 1 — Getter and Setter Binding

### The Basic Interception Pattern

The first step to building reactivity is simply **redirecting reads and writes** through custom functions. The actual data is stored in one object (`data`), but all access goes through a second object (`newData`) that has the custom getter/setter attached.

```javascript
// ─── Step 1: The raw data storage ───────────────────────────────────
const data = {
  count: 10
};

// ─── Step 2: The reactive proxy object ───────────────────────────────
const newData = {};

// ─── Step 3: Intercept 'count' on newData ────────────────────────────
Object.defineProperty(newData, 'count', {
  get() {
    return data.count;          // Read from the real storage
  },
  set(newValue) {
    data.count = newValue;      // Write to the real storage
  },
});

// ─── Usage ────────────────────────────────────────────────────────────
console.log(newData.count);   // → 10  (getter runs, reads from data.count)
newData.count = 20;           // setter runs, writes to data.count
console.log(newData.count);   // → 20  (getter runs, reads updated data.count)
console.log(data.count);      // → 20  (the underlying storage was updated)
```

**What just happened?**

```
Code writes: newData.count = 20
                    │
                    ▼  (intercepted by Object.defineProperty setter)
             set(newValue = 20)
                    │
                    ▼
             data.count = 20   (stored in the actual backing object)


Code reads:  newData.count
                    │
                    ▼  (intercepted by Object.defineProperty getter)
             get()
                    │
                    ▼
             return data.count  → 20
```

At this point, reads and writes are intercepted — but we haven't *done anything useful* with that interception yet. That's Step 2.

---

### Making It Generic — Reactifying Any Object

Real Vue doesn't do this for one property at a time. It loops over all properties of the `data()` return value and reactifies each one:

```javascript
function defineReactive(obj, key, value) {
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get() {
      return value;           // closure over 'value'
    },
    set(newValue) {
      if (newValue === value) return;  // no change — skip
      value = newValue;               // update the closed-over value
    }
  });
}

function observe(dataObj) {
  Object.keys(dataObj).forEach(key => {
    defineReactive(dataObj, key, dataObj[key]);
  });
}

// Usage
const state = {
  count: 10,
  username: 'rajesh',
  isLoggedIn: false
};

observe(state);

console.log(state.count);    // → 10  (getter)
state.count = 99;            // setter
console.log(state.count);    // → 99  (getter)
```

Each property now has its own closure-scoped `value` variable that the getter reads from and the setter writes to. No separate backing `data` object needed.

---

## 4.4 Implementation Step 2 — Track (get) and Trigger (set) Mechanism

### Adding the Notification Hooks

Now we add the actual reactivity intelligence: **registering dependencies on read** and **notifying them on write**.

```javascript
const data = {
  count: 10
};

const newData = {};

// ─── The two hook functions ───────────────────────────────────────────

function track() {
  // In a real system: record which component/watcher is currently
  // running and mark it as a dependency of this property
  console.log("Prop accessed");
}

function trigger() {
  // In a real system: look up all dependents of this property
  // and schedule them for re-execution (re-render or re-evaluate)
  console.log("Prop modified");
}

// ─── Intercept with track + trigger ──────────────────────────────────

Object.defineProperty(newData, 'count', {
  get() {
    track();              // 👈 Fire when READ
    return data.count;
  },
  set(newValue) {
    data.count = newValue;
    trigger();            // 👈 Fire when WRITTEN
  },
});

// ─── Demo ─────────────────────────────────────────────────────────────

console.log(newData.count);
// Output:
// Prop accessed        ← track() ran
// 10                   ← the actual value

newData.count = 20;
// Output:
// Prop modified        ← trigger() ran

console.log(newData.count);
// Output:
// Prop accessed        ← track() ran again
// 20                   ← the updated value
```

---

### What `track()` Really Does (Full Implementation)

In Vue's actual implementation, `track()` doesn't just print. It:

1. Checks if there is an **active "effect"** currently running (a render function or a watcher callback).
2. If yes, records: *"this property has a dependency on the currently running effect"*.
3. Stores this in a **dependency map** (`Map<property → Set<effects>>`).

```javascript
// Simplified but real-ish Vue 3 track implementation

let activeEffect = null;              // currently executing effect/watcher

// A Map: property key → Set of effects that depend on it
const dependencyMap = new Map();

function track(propertyKey) {
  if (!activeEffect) return;          // nothing is running, skip

  if (!dependencyMap.has(propertyKey)) {
    dependencyMap.set(propertyKey, new Set());
  }

  // Register activeEffect as a dependent of this property
  dependencyMap.get(propertyKey).add(activeEffect);

  console.log(`[track] '${propertyKey}' now has ${dependencyMap.get(propertyKey).size} subscriber(s)`);
}
```

---

### What `trigger()` Really Does (Full Implementation)

`trigger()` looks up the dependency map for the changed property and runs every registered effect:

```javascript
function trigger(propertyKey) {
  const effects = dependencyMap.get(propertyKey);

  if (!effects) return;               // no one depends on this — skip

  console.log(`[trigger] '${propertyKey}' changed — notifying ${effects.size} subscriber(s)`);

  // Re-run all effects that depend on this property
  effects.forEach(effect => effect());
}
```

---

### Putting It All Together — A Working Mini Reactivity System

Here is a minimal but complete reactivity system that replicates the core of Vue 2's approach:

```javascript
// ════════════════════════════════════════════════════════
//  MINI REACTIVITY SYSTEM
// ════════════════════════════════════════════════════════

let activeEffect = null;
const dependencyMap = new Map();

// ─── track: called in the getter ─────────────────────────
function track(key) {
  if (!activeEffect) return;
  if (!dependencyMap.has(key)) dependencyMap.set(key, new Set());
  dependencyMap.get(key).add(activeEffect);
}

// ─── trigger: called in the setter ───────────────────────
function trigger(key) {
  const deps = dependencyMap.get(key);
  if (deps) deps.forEach(effect => effect());
}

// ─── makeReactive: wraps any object ──────────────────────
function makeReactive(rawData) {
  const reactive = {};

  Object.keys(rawData).forEach(key => {
    let value = rawData[key];

    Object.defineProperty(reactive, key, {
      get() {
        track(key);          // Register dependency
        return value;
      },
      set(newValue) {
        if (newValue === value) return;
        value = newValue;
        trigger(key);        // Notify dependents
      }
    });
  });

  return reactive;
}

// ─── watchEffect: runs a function and re-runs on dependency change ───
function watchEffect(fn) {
  activeEffect = fn;
  fn();                      // Run once to collect dependencies (via track)
  activeEffect = null;
}

// ════════════════════════════════════════════════════════
//  USAGE DEMO
// ════════════════════════════════════════════════════════

const state = makeReactive({
  count: 0,
  username: 'rajesh'
});

// This simulates a component render function
watchEffect(() => {
  // Reading state.count inside this function causes track() to run
  console.log(`[RENDER] Count is now: ${state.count}`);
});
// → Immediately prints: [RENDER] Count is now: 0

state.count = 1;
// → trigger() fires → watchEffect re-runs → [RENDER] Count is now: 1

state.count = 2;
// → trigger() fires → watchEffect re-runs → [RENDER] Count is now: 2

state.username = 'kumar';
// → trigger() fires for 'username'
// → BUT watchEffect only read 'count', not 'username'
// → So watchEffect is NOT in username's dependency set
// → Nothing re-renders!  ✅ Efficient
```

**Complete execution trace**:
```
watchEffect(fn) called
  → activeEffect = fn
  → fn() executes
      → reads state.count   → track('count') → deps['count'] = { fn }
      → prints "[RENDER] Count is now: 0"
  → activeEffect = null

state.count = 1
  → setter runs → value = 1 → trigger('count')
  → trigger looks up deps['count'] = { fn }
  → fn() executes again
      → reads state.count   → track('count') → deps['count'] = { fn } (already there)
      → prints "[RENDER] Count is now: 1"

state.username = 'kumar'
  → setter runs → trigger('username')
  → deps['username'] = {} (empty Set — nobody read it)
  → nothing happens ✅
```

---

### Vue 2 vs Vue 3 Reactivity

Vue 3 (released 2020) switched from `Object.defineProperty()` to JavaScript `Proxy`. It's worth knowing why:

| Issue | `Object.defineProperty()` (Vue 2) | `Proxy` (Vue 3) |
|---|---|---|
| **Adding new properties** | ❌ Cannot detect (must use `Vue.set()`) | ✅ Automatically detected |
| **Deleting properties** | ❌ Cannot detect | ✅ Automatically detected |
| **Array mutations** | ❌ Can't intercept index assignment (`arr[0] = 1`) — Vue 2 patches array methods (`push`, `pop`, etc.) | ✅ All mutations detected |
| **Nested objects** | ❌ Must recursively walk entire object tree at init | ✅ Lazy — only reactifies on access |
| **Performance (large objects)** | ⚠️ Slow init (must pre-walk all properties) | ✅ Faster init (lazy proxy) |
| **Browser support** | IE9+ (very wide) | IE11+ only (modern browsers) |

**Vue 2 workaround for new properties** (because `Object.defineProperty` can't detect new property additions):
```javascript
// ❌ NOT reactive in Vue 2
this.user.newProp = 'value';

// ✅ Must use Vue.set() to make it reactive
this.$set(this.user, 'newProp', 'value');
// or
Vue.set(this.user, 'newProp', 'value');
```

**Vue 3 with Proxy** (everything just works):
```javascript
// ✅ Reactive automatically in Vue 3
const state = reactive({ user: {} });
state.user.newProp = 'value';   // Proxy intercepts this — reactive!
```

---

### Vue 3 Proxy — A Glimpse

```javascript
// Vue 3's reactive() simplified
function reactive(rawObject) {
  return new Proxy(rawObject, {
    get(target, key, receiver) {
      track(key);                          // Track on read
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver);
      trigger(key);                        // Trigger on write
      return result;
    },
    deleteProperty(target, key) {
      const result = Reflect.deleteProperty(target, key);
      trigger(key);                        // Even deletions trigger!
      return result;
    }
  });
}
```

A `Proxy` wraps an entire object with a single handler — intercepting *all* operations on *any* property, including ones that don't exist yet. This solves all of Vue 2's limitations cleanly.

---

### The Full Reactivity Pipeline — End-to-End

Here's how everything ties together when you use Vue normally:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VUE REACTIVITY PIPELINE                          │
│                                                                     │
│  1. COMPONENT INIT                                                  │
│     └─► data() returns plain object                                 │
│     └─► Vue wraps it with Object.defineProperty / Proxy            │
│         (every property now has custom get/set)                     │
│                                                                     │
│  2. COMPONENT RENDER                                                │
│     └─► render function executes                                    │
│     └─► template reads reactive data (e.g., this.count)            │
│     └─► getter fires → track() → component registered as dep       │
│     └─► Virtual DOM produced                                        │
│                                                                     │
│  3. DATA MUTATION                                                   │
│     └─► User action / timer / API response changes state           │
│     └─► setter fires → value updated → trigger()                   │
│     └─► trigger() looks up all dependents of that property         │
│     └─► Schedules dependent component re-renders                   │
│                                                                     │
│  4. RE-RENDER                                                       │
│     └─► Vue batches updates (async, next tick)                      │
│     └─► Renders new Virtual DOM                                     │
│     └─► Diffs against previous Virtual DOM                         │
│     └─► Patches only changed real DOM nodes                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Summary

| Concept | Mechanism | Key Point |
|---|---|---|
| **Reactivity basis** | Property interception | Must detect GET (read) and SET (write) |
| **Vue 2 approach** | `Object.defineProperty()` | Attaches getter/setter descriptors to each property |
| **Vue 3 approach** | `Proxy` | Wraps entire object; intercepts all operations including new props/deletions |
| **`track()`** | Called in the getter | Records which effect/component depends on this property |
| **`trigger()`** | Called in the setter | Re-runs all effects/components that depend on this property |
| **Dependency map** | `Map<key → Set<effects>>` | The data structure linking properties to their subscribers |
| **`activeEffect`** | Global context variable | Identifies the currently-running render/watcher during tracking |
| **`watchEffect`** | Auto-tracks dependencies | Runs fn immediately; re-runs whenever any read dependency changes |
| **Vue 2 `Vue.set()` limitation** | `Object.defineProperty` can't detect new properties | Must use `$set()` to add new reactive properties |
| **Batching** | Async update queue | Vue collects all triggers in a tick and applies them together for performance |

### Reactivity in Two Lines of Philosophy

```
GET  →  track()   →  "Who is reading this? Record them as a dependent."
SET  →  trigger() →  "This changed. Notify all recorded dependents."
```

Everything else in Vue's reactivity system — computed properties, watchers, component re-renders — is built on top of these two primitive operations.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

