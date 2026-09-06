# 5. Vue.js Component Lifecycle & Async Integration



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **5. Vue.js Component Lifecycle & Async Integration**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 5.1 Component Lifecycle Stages & Hooks

### The Mental Model: Birth, Life, and Teardown

Every Vue component instance goes through a predictable **lifecycle** — a series of stages from the moment it is created until the moment it is destroyed. At each stage transition, Vue automatically calls special **lifecycle hook functions** that you can define inside your component to run custom code at the right moment.

Think of it as a component's lifespan:

```
┌────────────────────────────────────────────────────────────────────┐
│                    COMPONENT LIFECYCLE                              │
│                                                                     │
│   BIRTH           LIFE                               TEARDOWN       │
│  ┌──────┐     ┌──────────────┐     ┌──────────┐   ┌────────────┐  │
│  │CREATE│────►│    MOUNT     │────►│  UPDATE  │──►│  DESTROY   │  │
│  └──────┘     └──────────────┘     └──────────┘   └────────────┘  │
│                                    (repeats on                      │
│                                     data changes)                   │
└────────────────────────────────────────────────────────────────────┘
```

Vue provides **two hooks per phase** — one called *before* the phase action and one called *after* — giving you precise control over when your code runs relative to Vue's internal work.

---

### Phase 1: Creation — `beforeCreate` & `created`

This is the very first phase. The component instance is being initialized.

```
new Vue() / component instantiated
        │
        ▼
  ┌─────────────┐
  │ beforeCreate│  ← Instance exists, but nothing is set up yet
  └─────────────┘  (No reactive data, no computed, no watchers, no events)
        │
        │  Vue initializes:
        │  ✔ Reactive data system (data(), props, computed)
        │  ✔ Watchers and methods
        │  ✔ Custom events ($emit/$on)
        │
        ▼
  ┌─────────────┐
  │   created   │  ← Reactivity is fully operational
  └─────────────┘  (Can access this.data, this.methods, this.computed)
                   (DOM does NOT exist yet — no $el, no template rendered)
```

#### `beforeCreate`
- Vue instance exists as an empty object
- `data()`, `computed`, `methods`, and `watch` are **not yet initialized**
- Rarely used in practice — almost nothing is accessible
- Used in some advanced plugin systems that need to hook in before reactivity

#### `created`
- Reactive data system is **fully active** — `this.myData`, `this.myMethod()` all work
- API calls can be initiated here (data will exist to store the results)
- The **DOM has not been created** — `this.$el` is `undefined`, `document.querySelector` won't find any component elements
- ✅ **The recommended hook for initiating data fetching** (explained in 5.2)

```js
export default {
    data() {
        return { user: null };
    },

    beforeCreate() {
        // ⚠️ this.user is undefined here — reactive data not yet initialized
        console.log(this.user);    // undefined
    },

    created() {
        // ✅ Reactive data is ready
        console.log(this.user);    // null (the initialized value)
        // Safe to start fetching data here
    }
};
```

---

### Phase 2: Mounting — `beforeMount` & `mounted`

Vue compiles the template and inserts the component's DOM into the page.

```
  created
     │
     │  Vue:
     │  ✔ Compiles template (or render function) into a Virtual DOM tree
     │  ✔ Begins the process of inserting into the real DOM
     │
     ▼
  ┌─────────────┐
  │ beforeMount │  ← Virtual DOM ready, but NOT yet inserted into real DOM
  └─────────────┘
     │
     │  Vue:
     │  ✔ Creates actual DOM elements from the Virtual DOM
     │  ✔ Inserts them into the document (replaces the mount point)
     │  ✔ this.$el now points to the live DOM element
     │
     ▼
  ┌─────────────┐
  │   mounted   │  ← Component is visible in the browser! Real DOM exists.
  └─────────────┘  (this.$el, this.$refs, document.querySelector all work now)
```

#### `beforeMount`
- The template has been compiled but the real DOM is not yet created
- Rarely used in practice
- `this.$el` does not yet exist

#### `mounted`
- The component's DOM is **live in the browser** — fully rendered and visible
- `this.$el` is now the root DOM element
- `this.$refs` (template refs) are accessible
- ✅ Required for any code that **needs to interact with the DOM** — charts, canvas, third-party widgets, measuring element dimensions

```js
export default {
    mounted() {
        // ✅ DOM exists here
        console.log(this.$el);               // The root DOM element
        console.log(this.$refs.myCanvas);    // A <canvas ref="myCanvas"> element

        // Third-party libraries that require a DOM element
        this.chart = new Chart(this.$refs.myCanvas, { type: 'bar', data: this.chartData });

        // Measuring DOM element dimensions
        console.log(this.$el.offsetWidth);   // Actual rendered width in pixels
    }
};
```

---

### Phase 3: Updating — `beforeUpdate` & `updated`

Whenever reactive data changes, Vue re-renders the component. This phase can repeat **many times** during a component's life.

```
  mounted
     │
     │  (User interaction, API response, or parent prop change causes data to change)
     │
     ▼
  ┌──────────────┐
  │ beforeUpdate │  ← Reactive data has changed, but DOM still shows old values
  └──────────────┘  (Good for reading the current DOM state before it changes)
     │
     │  Vue:
     │  ✔ Runs the Virtual DOM diffing algorithm (compares old vs new Virtual DOM)
     │  ✔ Patches only the changed DOM nodes (minimal real DOM operations)
     │
     ▼
  ┌─────────────┐
  │   updated   │  ← DOM has been updated to reflect new reactive data
  └─────────────┘  (Avoid mutating data here — could trigger an infinite update loop!)
```

#### `beforeUpdate`
- Reactive state has changed but the DOM still reflects the **old** values
- Useful for capturing the current scroll position or DOM measurements before a re-render

#### `updated`
- DOM is now synchronized with the new reactive state
- ⚠️ **Do not mutate reactive data here** — it would trigger another update, creating an infinite loop
- Use `nextTick()` if you need to perform DOM operations after a specific update

```js
export default {
    data() { return { count: 0 }; },

    beforeUpdate() {
        // DOM still shows old count value
        console.log('DOM count:', this.$el.textContent);   // Old value
        console.log('Data count:', this.count);             // New value
    },

    updated() {
        // DOM now reflects the new count
        console.log('DOM updated. New count in DOM:', this.$el.textContent);
        // ⚠️ DO NOT do: this.count++ — infinite loop!
    }
};
```

---

### Phase 4: Teardown / Destruction — `beforeUnmount` & `unmounted`

When a component is removed from the DOM (e.g., `v-if` becomes `false`, or the user navigates away), Vue tears it down.

> **Vue 2 names**: `beforeDestroy` / `destroyed`
> **Vue 3 names**: `beforeUnmount` / `unmounted` *(both sets are shown in this course)*

```
  mounted / updated
     │
     │  (v-if="false", router navigation, or parent removes component)
     │
     ▼
  ┌────────────────┐
  │ beforeUnmount  │  ← Component is still fully functional (Vue 3)
  │ (beforeDestroy)│     Watchers, events, child components all still active
  └────────────────┘     ✅ Last chance to do cleanup work
     │
     │  Vue:
     │  ✔ Tears down all watchers
     │  ✔ Removes all child component instances
     │  ✔ Removes all event listeners registered by Vue
     │
     ▼
  ┌───────────────┐
  │   unmounted   │  ← Component is gone from the DOM (Vue 3)
  │  (destroyed)  │     Most Vue functionality is torn down
  └───────────────┘     (Manual cleanup like timers/global listeners must be done in beforeUnmount)
```

```js
export default {
    data() {
        return { intervalId: null };
    },

    mounted() {
        // Start a timer when component mounts
        this.intervalId = setInterval(() => {
            console.log('Polling server...');
        }, 5000);
    },

    beforeUnmount() {   // Vue 3  (use beforeDestroy for Vue 2)
        // ✅ Clean up the timer before the component is destroyed
        clearInterval(this.intervalId);
        console.log('Timer cleared — component being removed');
    }
};
```

---

### Complete Lifecycle Hook Reference Table

| Hook | Vue 2 Name | Vue 3 Name | DOM Available? | When to Use |
|---|---|---|---|---|
| `beforeCreate` | `beforeCreate` | `beforeCreate` | ❌ No | Plugin internals; rarely used |
| `created` | `created` | `created` | ❌ No | **Initiate API calls**, set up non-DOM state |
| `beforeMount` | `beforeMount` | `beforeMount` | ❌ No | Rarely needed |
| `mounted` | `mounted` | `mounted` | ✅ Yes | Access `$el`, `$refs`; init third-party DOM libs (charts, maps) |
| `beforeUpdate` | `beforeUpdate` | `beforeUpdate` | ✅ Yes (old) | Read DOM before re-render |
| `updated` | `updated` | `updated` | ✅ Yes (new) | Post-render DOM operations (carefully!) |
| `beforeDestroy` / `beforeUnmount` | `beforeDestroy` | `beforeUnmount` | ✅ Yes | **Cleanup**: clear timers, remove listeners, cancel requests |
| `destroyed` / `unmounted` | `destroyed` | `unmounted` | ❌ No | Post-teardown logging; rarely needed |

---

## 5.2 Initiating Async Data Fetching in the Lifecycle

### Where to Trigger API Calls: `created()` vs `mounted()`

This is one of the most common practical questions when building Vue components that fetch data.

---

### Why `created()` Is Preferred for Pure Data Fetching

```js
export default {
    data() {
        return {
            posts: [],
            isLoading: true,
            error: null
        };
    },

    async created() {
        // ✅ Preferred: API call starts as early as possible
        // Reactive data (this.posts, this.isLoading) is already available
        // DOM doesn't need to exist for a network request
        try {
            const response = await axios.get('/api/posts');
            this.posts = response.data;
        } catch (err) {
            this.error = err.message;
        } finally {
            this.isLoading = false;
        }
    }
};
```

**Why `created()` is better for data fetching:**
- `created()` fires **before** the component is mounted to the DOM
- The network request starts sooner → data arrives sooner → the user waits less
- By the time Vue has finished mounting the component to the DOM, the API request is already in flight (or even complete for fast APIs)
- The reactive data system (`this.posts`, `this.isLoading`) is fully ready — no DOM is needed to store data

```
Timeline with created():
t=0ms  created() fires → axios.get() starts (request in-flight)
t=2ms  mounted() fires → DOM is inserted, shows "Loading..."
t=150ms API response arrives → this.posts updated → Vue re-renders → user sees data

Timeline with mounted():
t=0ms  created() fires → (nothing happens)
t=2ms  mounted() fires → DOM inserted, shows "Loading..." → axios.get() starts (request in-flight)
t=154ms API response arrives → user sees data
(~4ms slower — small here, but matters for slow network conditions)
```

---

### When `mounted()` Is Required

Some code **must** run in `mounted()` because it requires the DOM to exist:

```js
export default {
    data() { return { chartData: null }; },

    async created() {
        // ✅ Data fetching starts here (doesn't need DOM)
        const response = await axios.get('/api/chart-data');
        this.chartData = response.data;
    },

    mounted() {
        // ✅ Chart initialization MUST be here — needs a real canvas DOM element
        this.chart = new Chart(this.$refs.myChart, {
            type: 'line',
            data: this.chartData
        });

        // ✅ Measuring element width — needs real rendered DOM
        console.log('Component width:', this.$el.offsetWidth);

        // ✅ Third-party map library — needs a real div to attach to
        this.map = L.map(this.$refs.mapContainer).setView([19.0760, 72.8777], 13);
    }
};
```

| Scenario | Use `created()` | Use `mounted()` |
|---|---|---|
| Fetching API data to store in `data()` | ✅ Yes | ❌ Unnecessarily late |
| Initializing a chart (Chart.js, D3) | ❌ DOM not ready | ✅ Yes |
| Accessing `this.$refs.myElement` | ❌ Refs don't exist | ✅ Yes |
| Starting a `setInterval` for polling | ✅ Works | ✅ Also works |
| Measuring element dimensions (`offsetWidth`) | ❌ No DOM | ✅ Yes |
| Setting up a WebSocket connection | ✅ Works | ✅ Also works |

---

### Managing UI Async States: The 3-State Pattern

Whenever an async operation is in progress, the UI must communicate status to the user. The standard Vue pattern uses **three reactive data properties**:

```js
data() {
    return {
        data: null,        // Holds the result when fetch succeeds
        isLoading: true,   // true while the request is in-flight
        error: null        // Holds the error message if fetch fails
    };
}
```

These map to **three distinct UI states**:

```html
<template>
  <div>
    <!-- State 1: Loading -->
    <div v-if="isLoading" class="spinner">
      ⏳ Loading articles...
    </div>

    <!-- State 2: Error -->
    <div v-else-if="error" class="error-banner">
      ❌ {{ error }}
      <button @click="retry">Try Again</button>
    </div>

    <!-- State 3: Data ready -->
    <div v-else>
      <article v-for="post in posts" :key="post.id">
        <h2>{{ post.title }}</h2>
        <p>{{ post.body }}</p>
      </article>
    </div>
  </div>
</template>
```

The `finally` block is critical — it ensures `isLoading` becomes `false` whether the request succeeded or failed:

```js
async created() {
    // isLoading starts as true
    try {
        const res = await axios.get('/api/posts');
        this.posts = res.data;      // Success → populate data
    } catch (err) {
        this.error = err.message;   // Failure → store error message
    } finally {
        this.isLoading = false;     // Always runs → hide spinner
    }
}
```

```
State Machine:
  [isLoading=true, data=null, error=null]   ← Initial / Loading
          │
          ├── Success ──► [isLoading=false, data=[...], error=null]   ← Show data
          │
          └── Failure ──► [isLoading=false, data=null, error="..."]   ← Show error
```

---

## 5.3 Lifecycle Cleanup & Preventing Memory Leaks

### Why Cleanup Matters

When a component is **unmounted** (e.g., the user navigates to a different page via Vue Router, or a `v-if` removes it), Vue automatically cleans up:
- Its own watchers
- Its own event listeners (registered via `v-on` / `@`)
- Its child component instances

However, Vue **does NOT automatically clean up** things you set up manually:
- `setInterval()` / `setTimeout()` timers
- Global `window` or `document` event listeners
- Pending `fetch()` / `axios` requests
- WebSocket connections

If these are not cleaned up, they continue running after the component is gone — leaking memory and potentially causing errors when their callbacks try to update `this.someData` on a component instance that no longer exists.

---

### Cleanup 1: Clearing Intervals and Timeouts

```js
export default {
    data() {
        return {
            livePrice: null,
            pollInterval: null
        };
    },

    mounted() {
        // Start polling every 5 seconds
        this.pollInterval = setInterval(async () => {
            const res = await axios.get('/api/stock-price');
            this.livePrice = res.data.price;   // ⚠️ If component unmounts during fetch,
                                                //    this line runs on a dead instance!
        }, 5000);
    },

    beforeUnmount() {   // Vue 3 (beforeDestroy in Vue 2)
        // ✅ Stop the interval — no more callbacks will fire
        clearInterval(this.pollInterval);
    }
};
```

---

### Cleanup 2: Removing Global Event Listeners

Vue's `v-on` / `@` directive automatically removes listeners when the component unmounts. But `window.addEventListener()` and `document.addEventListener()` are **not managed by Vue** — they must be removed manually.

```js
export default {
    methods: {
        handleResize() {
            this.windowWidth = window.innerWidth;
        },
        handleKeydown(event) {
            if (event.key === 'Escape') this.closeModal();
        }
    },

    mounted() {
        // ⚠️ These are global — Vue doesn't know about them
        window.addEventListener('resize', this.handleResize);
        document.addEventListener('keydown', this.handleKeydown);
    },

    beforeUnmount() {
        // ✅ Must remove manually — using the EXACT same function reference
        window.removeEventListener('resize', this.handleResize);
        document.removeEventListener('keydown', this.handleKeydown);
    }
};
```

> **Critical**: `removeEventListener` requires the **exact same function reference** used in `addEventListener`. This is why the handler is defined as a named `method` rather than an inline arrow function — anonymous functions cannot be removed.

---

### Cleanup 3: Canceling Pending HTTP Requests

If a component is unmounted while an HTTP request is still in-flight, the response may arrive after the component is gone. The callback will then try to mutate data on a dead component instance, causing Vue warnings and potential memory leaks.

The solution is to **cancel the request** during teardown.

#### Canceling `fetch()` with `AbortController`

```js
export default {
    data() {
        return { results: [], controller: null };
    },

    async created() {
        // Create an AbortController and store it
        this.controller = new AbortController();

        try {
            const response = await fetch('/api/large-dataset', {
                signal: this.controller.signal    // Link the fetch to the controller
            });

            if (!response.ok) throw new Error('Request failed');
            this.results = await response.json();
        } catch (err) {
            if (err.name === 'AbortError') {
                console.log('Fetch was cancelled — component unmounted');
                // Don't treat a deliberate abort as an error
            } else {
                console.error('Fetch error:', err);
            }
        }
    },

    beforeUnmount() {
        // ✅ Cancel the in-flight fetch request
        if (this.controller) {
            this.controller.abort();
        }
    }
};
```

#### Canceling `axios` with `AbortSignal` (Modern Approach — Vue 3 / Axios 0.22+)

```js
import axios from 'axios';

export default {
    data() {
        return { data: null, controller: null };
    },

    async created() {
        this.controller = new AbortController();

        try {
            const response = await axios.get('/api/data', {
                signal: this.controller.signal    // Axios also accepts AbortSignal
            });
            this.data = response.data;
        } catch (err) {
            if (axios.isCancel(err) || err.name === 'AbortError') {
                console.log('Request cancelled');
            } else {
                console.error('Request failed:', err.message);
            }
        }
    },

    beforeUnmount() {
        // ✅ Cancel the in-flight axios request
        if (this.controller) {
            this.controller.abort();
        }
    }
};
```

---

### Complete Component: All Cleanup Patterns Together

```js
export default {
    data() {
        return {
            posts: [],
            isLoading: true,
            error: null,
            windowWidth: window.innerWidth,
            pollIntervalId: null,
            abortController: null
        };
    },

    async created() {
        // Set up AbortController for the initial fetch
        this.abortController = new AbortController();

        try {
            const res = await axios.get('/api/posts', {
                signal: this.abortController.signal
            });
            this.posts = res.data;
        } catch (err) {
            if (err.name !== 'AbortError') {
                this.error = err.message;
            }
        } finally {
            this.isLoading = false;
        }
    },

    mounted() {
        // Global event listener
        window.addEventListener('resize', this.handleResize);

        // Polling interval
        this.pollIntervalId = setInterval(this.refreshPosts, 30000);
    },

    methods: {
        handleResize() {
            this.windowWidth = window.innerWidth;
        },
        async refreshPosts() {
            const res = await axios.get('/api/posts');
            this.posts = res.data;
        }
    },

    beforeUnmount() {
        // ✅ Cancel any in-flight request
        this.abortController?.abort();

        // ✅ Remove global event listener
        window.removeEventListener('resize', this.handleResize);

        // ✅ Clear the polling interval
        clearInterval(this.pollIntervalId);
    }
};
```

---

## Summary

```
Topic 5: Vue.js Component Lifecycle & Async Integration
│
├── 5.1 Lifecycle Stages & Hooks
│     ├── Mental Model ──► Birth (Create) → Life (Mount → Update → ...) → Teardown (Destroy)
│     │
│     ├── Creation Phase
│     │     ├── beforeCreate ──► Instance exists, NO reactive data yet; rarely used
│     │     └── created      ──► Reactive data ready; NO DOM yet; ✅ ideal for API calls
│     │
│     ├── Mounting Phase
│     │     ├── beforeMount  ──► Template compiled, real DOM not yet inserted
│     │     └── mounted      ──► DOM live in browser; $el & $refs available; ✅ for DOM libs
│     │
│     ├── Updating Phase (repeats on data changes)
│     │     ├── beforeUpdate ──► Data changed, DOM still old; read pre-update DOM state
│     │     └── updated      ──► DOM patched; ⚠️ don't mutate data here (infinite loop)
│     │
│     └── Teardown Phase
│           ├── beforeUnmount ──► Component still functional; ✅ do all cleanup here
│           └── unmounted     ──► Component fully gone
│
├── 5.2 Async Data Fetching in the Lifecycle
│     ├── created() preferred ──► Earlier start, no DOM needed for network requests
│     ├── mounted() required  ──► DOM libs (charts, maps), $refs, offsetWidth
│     └── 3-State Pattern     ──► isLoading / error / data → Loading / Error / Content UI
│
└── 5.3 Lifecycle Cleanup & Memory Leak Prevention
      ├── Intervals/Timeouts  ──► clearInterval() / clearTimeout() in beforeUnmount
      ├── Global listeners    ──► removeEventListener() with same function reference
      └── HTTP Requests       ──► AbortController.abort() → fetch (signal) + axios (signal)
```

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
