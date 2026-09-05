# Module 1: State Management

---

## 1. Fundamentals of UI State

### 1.1 What is "State"?

**State** is any data that determines what the user sees on screen at a given moment. It encompasses:

- **Application data**: the list of products fetched from an API, the currently logged-in user's profile
- **UI flags**: `isModalOpen`, `isLoading`, `isSidebarCollapsed`
- **User input in-progress**: text typed in a search box before it is submitted
- **Selections & toggles**: active tab index, selected items in a list, accordion open/closed

Every visible element of a web page is ultimately a **function of some state**.

### 1.2 The Declarative Paradigm: $\text{UI} = f(\text{State})$

In **imperative** programming (vanilla JS, jQuery), the developer manually manipulates the DOM to reflect data changes:

```javascript
// Imperative: developer manually syncs DOM to data
let cartCount = 0;

function addToCart(item) {
  cartCount++;
  document.getElementById('cart-badge').innerText = cartCount;
  document.getElementById('cart-badge').style.display = 'block';
  if (cartCount > 9) {
    document.getElementById('cart-badge').innerText = '9+';
  }
  // ... more manual DOM updates
}
```

Problems:
- DOM manipulation code is scattered everywhere
- As the app grows, keeping every DOM node in sync with every data change becomes unmanageable
- A single forgotten `document.getElementById` call means the UI is out of sync with the data

In **declarative** frameworks like Vue, the UI is defined as a **pure function of state**:

$$\mathbf{UI} = f(\mathbf{State})$$

```html
<!-- Declarative: describe WHAT the UI should look like given the state -->
<span class="cart-badge" v-if="cartCount > 0">
  {{ cartCount > 9 ? '9+' : cartCount }}
</span>
```

```javascript
data() {
  return { cartCount: 0 };
},
methods: {
  addToCart(item) {
    this.cartCount++;     // Only mutate state — Vue handles the DOM automatically
  }
}
```

**How it works:**
- $\text{State}$: the reactive data (`cartCount`, `user`, `products`, etc.)
- $f$: Vue's template / render function
- $\text{UI}$: the resulting DOM nodes on screen

When `cartCount` changes, Vue automatically re-evaluates $f(\text{State})$, computes a **Virtual DOM diff**, and applies the minimal necessary DOM updates. The developer never touches the DOM.

### 1.3 Why Declarative State is Powerful

```
Imperative approach:
  State changes → Developer manually identifies every affected DOM node
                → Developer writes code to update each one
                → Easy to miss updates, create bugs, have stale UI

Declarative approach:
  State changes → Vue automatically re-renders the entire affected subtree
                → UI is ALWAYS an accurate reflection of state
                → Developer only thinks about data, never DOM
```

---

## 2. The State Management Pattern & One-Way Data Flow

### 2.1 The Three Pillars

A self-contained Vue component manages its own internal state through three interconnected concepts:

```
┌────────────────────────────────────────────┐
│                                            │
│   STATE                                    │
│   The source of truth. Raw data.           │
│   data() { return { count: 0 } }           │
│            │                               │
│            │  f(State) — declarative mapping│
│            ▼                               │
│   VIEW                                     │
│   What the user sees. Rendered HTML.       │
│   <p>{{ count }}</p>                       │
│   <button @click="increment">+</button>    │
│            │                               │
│            │  user interaction             │
│            ▼                               │
│   ACTIONS                                  │
│   What happens when user interacts.        │
│   methods: { increment() { this.count++ } }│
│            │                               │
│            │  mutates State                │
│            └───────────────────────────────┤
│                                            │
└────────────────────────────────────────────┘
```

1. **State**: The single authoritative source of truth. All reactive data that drives the UI.
2. **View**: The declarative template rendered from the current state. Pure output — no logic.
3. **Actions**: Event handlers and methods triggered by user interaction. They update the state.

### 2.2 One-Way (Unidirectional) Data Flow

Data moves in **one direction only**, completing a cycle:

```
         ┌─────────┐
         │  State  │◄──────────────────────┐
         └────┬────┘                       │
              │ renders                    │
              ▼                            │ mutates
         ┌─────────┐                  ┌───┴────┐
         │  View   │──── triggers ───►│Actions │
         └─────────┘                  └────────┘
```

**Why unidirectional?**

In older frameworks (e.g., classic Angular 1 two-way binding at scale), data could flow in both directions freely. This made it extremely difficult to trace *who changed what* when a bug appeared. With one-way flow:

- A state change always starts from an explicit Action
- Actions always flow through defined, named methods
- The chain is predictable and traceable at every step

**Concrete example:**

```
1. State: { count: 0 }
   View renders: "Count: 0" and a "+" button

2. User clicks "+"
   Action: increment() is called

3. increment() mutates: this.count++
   State is now: { count: 1 }

4. Vue detects reactive state changed
   View re-renders: "Count: 1"
   → back to step 1 with new state
```

---

## 3. UI State vs. Server State — Contrast with MVC

### 3.1 A Common Confusion

Students coming from backend-focused courses often ask: *"We already have state in the database and the MVC model — why do we need state management on the frontend too?"*

These are two entirely **different kinds of state** operating at different layers:

### 3.2 Comparison Table

| Dimension | **UI / Frontend State** | **System / Backend State (MVC)** |
| :--- | :--- | :--- |
| **Where it lives** | JavaScript memory in the browser | Database (PostgreSQL, MongoDB, etc.) |
| **Lifespan** | Exists only while browser tab is open | Persists indefinitely across sessions |
| **Examples** | `isDropdownOpen`, `activeTabIndex`, `searchQuery`, `formDraft` | User records, orders, product inventory |
| **Managed by** | Vue, React, Vuex, Pinia | Flask/Django Model, SQLAlchemy ORM |
| **Reactivity** | Instant, sub-millisecond DOM updates | Request/response cycle (100ms–seconds) |
| **Scope** | One user's current browser session | All users, all sessions, all time |
| **Lost when** | User closes or refreshes the tab | Never (unless explicitly deleted) |

### 3.3 They Coexist — Not Competing

Frontend state management does **not** replace the backend MVC pattern:

```
Backend (Flask + SQLAlchemy MVC):
  Model:      User, Product, Order database tables
  View:       JSON API responses (not HTML anymore in SPAs)
  Controller: Route handlers, business validation, auth

Frontend (Vue + Vuex):
  State:      Cached API data, UI flags, current user session data
  View:       Vue templates rendered from state
  Actions:    User interactions → API calls → state mutations
```

A typical SPA flow combining both:

```
User clicks "Add to Cart"
      │
      ▼
Vue Action (frontend) → POST /api/cart { productId: 5 }
      │                         │
      │                         ▼
      │                 Flask Controller validates request
      │                 SQLAlchemy updates CartItem in DB
      │                 Returns JSON { success: true, cartTotal: 3 }
      │                         │
      ▼                         │
Vue Mutation (frontend) receives response
Commits: SET_CART_COUNT(3)
      │
      ▼
Vue re-renders cart badge: shows "3"
```

---

## 4. Component Hierarchy & Communication Challenges

### 4.1 How Components Communicate (The Right Way)

Vue's component system enforces **explicit, directional data flow**:

```
             Parent Component
             data: { title: 'Hello' }
                    │              ▲
       props down   │              │  events up
    :title="title"  │              │  $emit('update', value)
                    ▼              │
             Child Component
             props: ['title']
```

#### Parent → Child: Props

```html
<!-- Parent template -->
<UserCard :user="currentUser" :is-admin="userIsAdmin" />
```

```javascript
// Child component (UserCard.vue)
export default {
  props: {
    user: { type: Object, required: true },
    isAdmin: { type: Boolean, default: false }
  }
}
```

Props are **read-only** inside the child. A child must never directly mutate a prop.

#### Child → Parent: Custom Events (`$emit`)

```javascript
// Child component: signals to parent that something happened
methods: {
  handleDeleteClick() {
    this.$emit('delete-user', this.user.id);
    // Sends event named 'delete-user' with payload this.user.id up to parent
  }
}
```

```html
<!-- Parent template: listens for the emitted event -->
<UserCard :user="currentUser" @delete-user="onDeleteUser" />
```

```javascript
// Parent component: handles the event
methods: {
  onDeleteUser(userId) {
    // Now parent can decide what to do with the user's deletion request
    this.users = this.users.filter(u => u.id !== userId);
  }
}
```

### 4.2 The Anti-Pattern: Direct Parent Access

A child *can* technically do:

```javascript
// Child component — NEVER do this
this.$parent.someData = 'mutated from child'; // ❌ Anti-pattern
this.$parent.someMethod(); // ❌ Anti-pattern
```

**Why this is dangerous:**
- **Tight coupling**: The child now depends on knowing the internal structure of its parent. Moving or reusing the child component anywhere else breaks the app.
- **Silent mutations**: The parent's state is modified without any visible event trail. When debugging, you have no idea who changed the data.
- **Breaks the one-way flow contract**: Data no longer flows predictably; it can change from anywhere.

### 4.3 The Sibling Problem: Prop Drilling & Event Bubbling

What happens when two components that are **not in a parent-child relationship** need to share state?

Consider this component tree:

```
                    App.vue
                 /          \
          Header.vue      MainContent.vue
          /                /           \
    SearchBar.vue    Sidebar.vue    ProductList.vue
                                         │
                                   ProductCard.vue
```

Scenario: User types a search query in `SearchBar.vue`. `ProductList.vue` needs to filter products based on that query.

**Without global state — the painful path:**

```
SearchBar emits 'search' to Header
Header emits 'search' to App
App passes :searchQuery prop to MainContent
MainContent passes :searchQuery prop to ProductList
ProductList finally uses the searchQuery to filter

= 4 layers of props/events for one piece of data
```

This is called **prop drilling** — threading a value through layers of intermediary components that don't actually use it themselves, just to deliver it to a deeply nested consumer.

**Problems as the app scales:**
- Every intermediary component must declare and pass through the prop/event even though it has no use for it
- Refactoring the component tree breaks the entire data delivery chain
- Adding a new consumer of the same data requires re-threading through the entire hierarchy again

---

## 5. Evaluating Global State Solutions

### 5.1 The Naive Solution: Global JavaScript Object

The obvious fix seems to be: put shared state somewhere globally accessible.

```javascript
// Naive approach: plain global object
window.appState = {
  searchQuery: '',
  cartItems: [],
  currentUser: null
};
```

Or as an exported module:

```javascript
// sharedState.js
export const sharedState = {
  searchQuery: '',
  cartItems: []
};
```

**Why this fails:**

| Problem | Explanation |
| :--- | :--- |
| **No reactivity** | Plain JS objects are not reactive. Changing `window.appState.searchQuery` does nothing to the Vue component — it won't re-render. |
| **No traceability** | Any component, method, or async callback can write `window.appState.cartItems = []` at any time. When a bug causes the cart to mysteriously empty, you cannot determine who did it or when. |
| **Race conditions** | Multiple async operations writing to the same global object simultaneously produce unpredictable results. |
| **No dev tooling** | There is no way to log, replay, or inspect these mutations. |

### 5.2 The Right Mental Model: Restricted Global Access

A proper global state solution must satisfy two requirements:

```
1. READ:    Any component anywhere can READ from the global state freely.
            (Components should be able to display global data without ceremony)

2. WRITE:   State can ONLY be modified through explicitly defined, named handlers.
            (No direct assignments: state.count++ is forbidden; only commit('increment') is allowed)
```

This design guarantees that every state change is:
- **Named**: you can see in logs that `INCREMENT_CART` happened
- **Trackable**: you know which action triggered it
- **Reversible**: you can replay or undo it in dev tools

This is exactly what **Vuex** implements.

---

## 6. Ecosystem Perspectives: Flux, Redux & Elm

Vuex was not invented in isolation. It drew from influential patterns that emerged to solve the same class of problems in other ecosystems.

### 6.1 The Problem That Motivated Flux (Facebook, 2014)

Facebook's engineering team discovered that their MVC architecture broke down at scale. With many models talking to many views bidirectionally, a single action could trigger cascading updates in unpredictable sequences — famously demonstrated by the notification counter bug that kept showing unread counts even when the inbox was empty.

**Their diagnosis**: Two-way data binding between many models and many views creates a spaghetti of dependencies impossible to reason about.

**Their solution**: Enforce strict unidirectional data flow.

### 6.2 Flux Architecture

```
┌──────────┐   action   ┌────────────┐   action   ┌───────┐   change event   ┌──────┐
│  Action  │──────────►│ Dispatcher │──────────►│ Store │────────────────►│ View │
│ Creators │            └────────────┘            └───────┘                  └──┬───┘
└──────────┘                                                                    │
     ▲                                                                          │ user interaction
     └──────────────────────────────────────────────────────────────────────────┘
```

- **Action Creators**: Functions that create action objects describing *what happened*
- **Dispatcher**: A central hub that broadcasts every action to every registered Store
- **Store**: Contains application state and the logic to respond to specific actions
- **View**: React components that subscribe to Store changes and re-render

The key insight: every mutation goes through the Dispatcher, creating a single observable pipeline.

### 6.3 Redux (Dan Abramov, 2015)

Redux distilled Flux further into three immutable principles:

**Principle 1 — Single Source of Truth:**
```
The entire application state is stored in a single JavaScript object tree
inside a single store. There is exactly ONE store.

state = {
  user: { id: 1, name: 'Alice' },
  cart: { items: [], total: 0 },
  ui: { isLoading: false, activeModal: null }
}
```

**Principle 2 — State is Read-Only:**
```
State cannot be directly assigned. The only way to change state
is to dispatch an Action — a plain object describing what happened.

store.dispatch({ type: 'INCREMENT_CART', payload: { productId: 5 } })

// You NEVER do:
store.state.cart.items.push(product) // ❌ Forbidden
```

**Principle 3 — Changes are Made with Pure Reducer Functions:**

A **reducer** is a pure function: `(previousState, action) → newState`

```javascript
function cartReducer(state = { items: [], total: 0 }, action) {
  switch (action.type) {
    case 'ADD_ITEM':
      return {
        ...state,               // never mutate previous state
        items: [...state.items, action.payload],
        total: state.total + action.payload.price
      };
    case 'CLEAR_CART':
      return { items: [], total: 0 };
    default:
      return state;             // always return state unchanged for unknown actions
  }
}
```

- **Pure**: given the same inputs, always produces the same output
- **No side effects**: never makes API calls, reads from disk, or modifies external variables
- **Immutable**: returns a new state object instead of mutating the old one

### 6.4 The Elm Architecture

Elm is a purely functional language that compiles to JavaScript. Its architecture later heavily influenced Redux:

```
┌────────────────────────────────────────────────┐
│                                                │
│   MODEL         The application state.         │
│   (State)       A single immutable record.     │
│       │                                        │
│       │  view function                         │
│       ▼                                        │
│   VIEW          A function: Model → HTML       │
│   (Template)    Pure — no side effects.        │
│       │                                        │
│       │  user interaction sends Msg            │
│       ▼                                        │
│   UPDATE        A function: (Msg, Model) → Model│
│   (Reducer)     Pure — returns new Model.      │
│       │                                        │
│       └────────────────────────────────────────┘
│
└────────────────────────────────────────────────┘
```

The Elm architecture guarantees that **no runtime exceptions can occur** (Elm has no `null`, no undefined is not a function) and that every state transition is perfectly trackable.

---

## 7. Vuex Architecture & Core Building Blocks

**Vuex** is the official, Vue-team-maintained library for centralized state management in Vue.js applications. It adapts the principles of Flux/Redux to work natively with Vue's reactivity system.

### 7.1 The Big Picture

```
                    ┌──────────────────────────────────┐
                    │          Vue Components           │
                    │                                  │
                    │  computed: {                     │
                    │    count() {                     │
                    │      return this.$store.state.count│
                    │    }                             │
                    │  }                               │
                    └──────┬──────────────────▲────────┘
                           │                  │
                 dispatch  │                  │  reactive render
                           ▼                  │
                    ┌──────────────┐           │
                    │   Actions    │           │
                    │  (async OK) │           │
                    └──────┬───────┘           │
                           │ commit            │
                           ▼                  │
                    ┌──────────────┐          │
                    │  Mutations   │  ────────►│  State
                    │ (sync only)  │  mutates  │  (reactive)
                    └──────────────┘          │
                                              │
                    ┌──────────────┐          │
                    │   Getters    │◄─────────┘
                    │ (computed    │  derived from
                    │  from state) │
                    └──────────────┘
```

### 7.2 The Four Core Concepts

| Concept | What it is | Analogy | How you access it |
| :--- | :--- | :--- | :--- |
| **State** | The raw reactive data object | The database | `this.$store.state.count` |
| **Getters** | Computed/derived values from state | Database views / computed columns | `this.$store.getters.evenCount` |
| **Mutations** | Synchronous functions that change state | Database write transactions | `this.$store.commit('mutationName')` |
| **Actions** | Async operations that eventually commit mutations | Database stored procedures with API calls | `this.$store.dispatch('actionName')` |

### 7.3 The Single State Tree

Vuex uses a **single state tree** — one JavaScript object that is the *only* authoritative representation of all application-level state:

```javascript
// ONE object represents the entire app's global state
const state = {
  currentUser: null,
  products: [],
  cart: {
    items: [],
    couponCode: null
  },
  ui: {
    isLoading: false,
    activeModal: null,
    sidebarOpen: true
  }
};
```

**Benefits:**
- Snapshots of the entire app state at any point in time are trivially simple
- Serializing state to JSON for debugging, bug reports, or time-travel is straightforward
- A single place to look when understanding "what is the app's current situation"

### 7.4 Local State vs. Global State — When to Use Each

Not everything belongs in the Vuex store. A useful decision rule:

```
Ask: "Do multiple unrelated components need this data?"

YES → Put it in Vuex (global)
  Examples:
  - Authenticated user's profile
  - Shopping cart items
  - List of products fetched from API
  - Global loading indicator
  - Application theme / dark mode preference

NO → Keep it as local component state
  Examples:
  - Whether a dropdown is open in a specific component
  - The current value of a text input before form submit
  - Which accordion panel is expanded
  - A hover state
```

### 7.5 Basic Vuex Store Setup

```javascript
// store/index.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.Store({

  // ── STATE ──────────────────────────────────────────────────
  state: {
    count: 0,
    user: null,
    products: []
  },

  // ── GETTERS ────────────────────────────────────────────────
  getters: {
    // Like computed properties for the store
    doubleCount(state) {
      return state.count * 2;
    },
    isLoggedIn(state) {
      return state.user !== null;
    },
    expensiveProducts(state) {
      return state.products.filter(p => p.price > 100);
    }
  },

  // ── MUTATIONS ──────────────────────────────────────────────
  mutations: {
    INCREMENT(state) {
      state.count++;
    },
    SET_USER(state, user) {
      state.user = user;
    },
    SET_PRODUCTS(state, products) {
      state.products = products;
    }
  },

  // ── ACTIONS ────────────────────────────────────────────────
  actions: {
    async fetchProducts({ commit }) {
      const res = await fetch('/api/products');
      const data = await res.json();
      commit('SET_PRODUCTS', data);
    }
  }

});

export default store;
```

```javascript
// main.js — inject store into root Vue instance
new Vue({
  store,  // makes this.$store available in every component
  render: h => h(App)
}).$mount('#app')
```

### 7.6 Accessing Store in Components

```html
<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Double: {{ doubleCount }}</p>
    <p>Status: {{ isLoggedIn ? 'Logged In' : 'Guest' }}</p>
    <button @click="increment">+1</button>
  </div>
</template>

<script>
export default {
  computed: {
    // Access state
    count()       { return this.$store.state.count; },
    // Access getters
    doubleCount() { return this.$store.getters.doubleCount; },
    isLoggedIn()  { return this.$store.getters.isLoggedIn; }
  },
  methods: {
    increment() {
      this.$store.commit('INCREMENT');
    }
  }
};
</script>
```

---

## 8. Vuex Mutations & Synchronous State Changes

### 8.1 The Golden Rule of Mutations

> **Mutations are the ONLY place in Vuex where state is allowed to be modified.**
> **Mutations MUST always be synchronous.**

These two rules together are what make Vuex's time-travel debugging possible.

### 8.2 Why Mutations Must Be Synchronous

When Vue Devtools logs a mutation, it captures:
- The mutation's name
- The payload passed to it
- A **snapshot of the state before** the mutation ran
- A **snapshot of the state after** the mutation ran

For this before/after snapshot to be meaningful, the state must change **predictably and immediately** when the mutation handler runs. If a mutation were allowed to contain asynchronous callbacks:

```javascript
// ❌ WRONG — async inside mutation
mutations: {
  SET_USER_ASYNC(state) {
    setTimeout(() => {
      state.user = { name: 'Alice' }; // This runs AFTER the mutation "completes"
    }, 1000);
  }
}
```

The Devtools would record the mutation as complete (capturing state before the `setTimeout` fires), then a second state change would happen silently inside the timer callback with no mutation logged — making the state history incorrect and useless for debugging.

### 8.3 Commit Patterns

#### 1. Basic Commit (No Payload)
```javascript
// Mutation definition
mutations: {
  INCREMENT(state) {
    state.count++;
  }
}

// Usage in component or action
this.$store.commit('INCREMENT')
```

#### 2. Commit with Payload (Primitive)
```javascript
// Mutation definition
mutations: {
  INCREASE_BY(state, amount) {
    state.count += amount;
  }
}

// Usage
this.$store.commit('INCREASE_BY', 10)
this.$store.commit('INCREASE_BY', 25)
```

#### 3. Commit with Payload (Object)
```javascript
// Mutation definition — object payload
mutations: {
  UPDATE_PROFILE(state, payload) {
    state.user.name = payload.name;
    state.user.email = payload.email;
  }
}

// Usage
this.$store.commit('UPDATE_PROFILE', {
  name: 'Alice',
  email: 'alice@example.com'
})
```

#### 4. Object-Style Commit
```javascript
// Alternative commit syntax where the entire argument is an object
// The `type` key is the mutation name
this.$store.commit({
  type: 'UPDATE_PROFILE',
  name: 'Alice',
  email: 'alice@example.com'
})

// The mutation receives the entire commit object as payload
mutations: {
  UPDATE_PROFILE(state, payload) {
    // payload = { type: 'UPDATE_PROFILE', name: 'Alice', email: 'alice@example.com' }
    state.user.name = payload.name;
    state.user.email = payload.email;
  }
}
```

---

## 9. Vue Devtools & Time-Travel Debugging

### 9.1 What the Devtools Record

Because every state change flows through a named, synchronous mutation, Vue Devtools can build a **complete, chronological audit log** of everything that happened to the application state:

```
Timeline of Mutations:

  [14:23:01.042]  FETCH_START          payload: null
                  state.isLoading: false → true

  [14:23:01.891]  SET_PRODUCTS         payload: [ { id:1, ... }, { id:2, ... }, ... ]
                  state.products: [] → [ 47 items ]

  [14:23:01.892]  FETCH_END            payload: null
                  state.isLoading: true → false

  [14:23:04.120]  ADD_TO_CART          payload: { productId: 3, quantity: 1 }
                  state.cart.items: [] → [{ id:3, qty:1 }]
```

For every mutation you can see: **when** it happened, **what** the payload was, **what the state looked like before**, and **what it looked like after**.

### 9.2 Time-Travel Debugging

The Devtools panel lists every mutation in sequence. You can:

1. **Click any past mutation** → The UI instantly reverts to the exact state at that point in time
2. **Step forward** mutation by mutation to watch the state evolve
3. **Commit** or **revert** individual mutations to see their isolated effect on the UI
4. **Export** the entire state tree as JSON and share it with a teammate to reproduce a bug
5. **Import** a state snapshot to instantly reproduce a reported bug state locally

This is only possible because mutations are synchronous, named, and pass through a single channel.

---

## 10. Vuex Actions & Asynchronous Logic

### 10.1 Why Actions Exist

Real applications need to:
- Fetch data from APIs before updating state
- Wait for a file upload to complete before committing progress
- Dispatch multiple mutations in sequence
- Run conditional logic based on the current state before mutating

None of this can live in mutations (synchronous only). **Actions** handle all asynchronous operations and complex business logic, ultimately committing mutations when they have a result.

### 10.2 Actions vs. Mutations

| | **Mutations** | **Actions** |
| :--- | :--- | :--- |
| **Can be async?** | ❌ No — must be synchronous | ✅ Yes — async/await, Promises |
| **Modifies state directly?** | ✅ Yes — only place that can | ❌ No — commits mutations only |
| **Triggered by** | `store.commit('name')` | `store.dispatch('name')` |
| **Purpose** | Atomic state write operation | Async coordination, business logic |

### 10.3 Action Syntax & Context Object

An action receives a **context** object as its first argument. Context exposes the same methods and properties as the store:

```javascript
actions: {
  // Full context object
  doSomething(context) {
    context.commit('MUTATION_NAME');
    context.dispatch('anotherAction');
    console.log(context.state.count);
    console.log(context.getters.doubleCount);
  },

  // Destructured (much more common in practice)
  async fetchUser({ commit, state, dispatch, getters }, userId) {
    commit('SET_LOADING', true);

    try {
      const res = await fetch(`/api/users/${userId}`);
      const user = await res.json();
      commit('SET_USER', user);
    } catch (error) {
      commit('SET_ERROR', error.message);
    } finally {
      commit('SET_LOADING', false);
    }
  }
}
```

### 10.4 Dispatching Actions from Components

```javascript
// In a Vue component
methods: {
  async loadUser() {
    await this.$store.dispatch('fetchUser', this.userId);
    // state is updated after this line
  },

  // Object-style dispatch
  login() {
    this.$store.dispatch({
      type: 'authenticate',
      email: this.email,
      password: this.password
    });
  }
}
```

### 10.5 Practical Pattern: Optimistic UI with Rollback

```javascript
actions: {
  async addToCart({ state, commit }, product) {
    // 1. Save current state for potential rollback
    const previousItems = [...state.cart.items];

    // 2. Optimistically update UI immediately (feels instant to user)
    commit('ADD_CART_ITEM', product);

    try {
      // 3. Confirm with server
      await fetch('/api/cart', {
        method: 'POST',
        body: JSON.stringify({ productId: product.id }),
        headers: { 'Content-Type': 'application/json' }
      });
      // Success: optimistic update was correct, nothing more to do

    } catch (error) {
      // 4. Server failed: rollback to previous state
      commit('SET_CART_ITEMS', previousItems);
      commit('SET_ERROR', 'Failed to add to cart. Please try again.');
    }
  }
}
```

### 10.6 Composing Actions — Chaining with async/await

`store.dispatch()` always returns a Promise. This allows actions to await other actions:

```javascript
actions: {
  async login({ commit, dispatch }, credentials) {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify(credentials)
    });
    const { token, userId } = await res.json();

    commit('SET_TOKEN', token);

    // Compose: wait for fetchUserProfile to complete before continuing
    await dispatch('fetchUserProfile', userId);
    // Now both token AND user profile are in state

    await dispatch('fetchUserCart', userId);
    // Now cart data is also loaded

    commit('SET_READY', true); // Signal that initialization is complete
  },

  async fetchUserProfile({ commit }, userId) {
    const res = await fetch(`/api/users/${userId}`);
    const profile = await res.json();
    commit('SET_USER', profile);
  },

  async fetchUserCart({ commit }, userId) {
    const res = await fetch(`/api/cart/${userId}`);
    const cart = await res.json();
    commit('SET_CART', cart);
  }
}
```

---

## 11. State Management Summary

### 11.1 When Should You Use Vuex?

```
Use Vuex when:
  ✅ Multiple unrelated components need to read the same data
  ✅ State needs to persist across route changes
  ✅ Async data fetching results need to be shared app-wide
  ✅ Complex workflows where multiple steps must happen in sequence
  ✅ You need time-travel debugging for complex bugs

Keep state local when:
  ✅ Only one component and its children care about the data
  ✅ The data is ephemeral UI state (hover, focus, open/closed)
  ✅ The data doesn't make sense outside that component's context
```

### 11.2 The Full Vuex Data Flow

```
User Interaction (click, submit, keypress)
           │
           ▼
   Vue Component Method
           │
           │ this.$store.dispatch('actionName', payload)
           ▼
   Vuex Action (async OK)
   ├── await API calls
   ├── await other actions
   └── this.$store.commit('MUTATION', data)
                │
                ▼
   Vuex Mutation (sync only)
   └── state.field = data  ← ONLY place state changes
                │
                ▼
   Vuex State (reactive)
                │
                │ Vue detects reactive change
                ▼
   Getters recompute (if dependent on changed state)
                │
                ▼
   Components re-render (computed properties update)
                │
                ▼
   Updated UI visible to user
```

### 11.3 Key Exam Rules to Memorize

| Rule | Detail |
| :--- | :--- |
| $\text{UI} = f(\text{State})$ | UI is always a pure reflection of state |
| One-way data flow | View → Action → State → View, never backwards |
| Mutations: sync only | Async code in mutations breaks Devtools time-travel |
| Actions: async allowed | Actions commit mutations when async work completes |
| Single state tree | One store object, one source of truth |
| `commit()` vs `dispatch()` | `commit` triggers mutations; `dispatch` triggers actions |
| Props down, events up | Standard component communication without Vuex |
| Vuex vs local state | Global = Vuex; ephemeral/single-component = `data()` |
