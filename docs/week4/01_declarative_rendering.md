# 1. Declarative Rendering

> **Module**: MAD II — Week 4 | **Topic**: Vue.js Fundamentals
> **Subtopics**: 1.1 → 1.8

---

## 1.1 Concept of Reactivity

### What is Reactivity?

Reactivity in the context of UI frameworks means that the **view (what you see on screen) automatically updates whenever the underlying data (state) changes** — without you having to manually manipulate the DOM.

Think of it like a spreadsheet: when you change a value in cell A1, every formula that depends on A1 instantly recalculates. Vue works the same way — change the data, the UI updates itself.

### The Paradigm Shift: Declarative vs Imperative

| Approach | Style | You describe... |
|---|---|---|
| **Imperative** (jQuery, vanilla JS) | "How" | Step-by-step DOM operations |
| **Declarative** (Vue) | "What" | The desired end state of the UI |

**Imperative (old way)**:
```javascript
// You have to manually select elements and update them
document.getElementById('username').textContent = user.name;
document.getElementById('avatar').src = user.avatarUrl;
document.getElementById('nav-login').style.display = 'none';
document.getElementById('nav-logout').style.display = 'block';
```

**Declarative (Vue way)**:
```html
<!-- Just describe what the UI should look like based on state -->
<span>{{ user.name }}</span>
<img :src="user.avatarUrl" />
<button v-if="!isLoggedIn">Login</button>
<button v-if="isLoggedIn">Logout</button>
```
When `user.name` changes, the `<span>` updates automatically. No manual DOM calls needed.

### Core Mechanism

Vue establishes a **reactive binding** between:
- The **Model** (your JavaScript data/state object)
- The **View** (the HTML template rendered in the browser)

When data changes → Vue detects it → re-renders only the affected parts of the DOM.

---

## 1.2 Why Reactivity? (Motivation & State Changes)

### The Problem: UI State is Complex

Modern web apps are highly interactive. A **single user action** can trigger a cascade of UI updates across multiple independent parts of the page simultaneously.

### Real-World Example: User Login

When a user logs in, the following things might need to update:

```
User clicks "Login" button
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  Things that need to update on screen                    │
│                                                          │
│  1. Navigation bar  →  show "Profile" & "Logout" links   │
│  2. Main content    →  show enrolled courses / dashboard │
│  3. Sidebar         →  show personalized recommendations │
│  4. Theme/Colors    →  apply user's saved preferences    │
│  5. Notifications   →  load pending alerts               │
└──────────────────────────────────────────────────────────┘
```

### Without Reactivity (Manual Approach):
You would write **imperative code** to touch each of these 5 DOM regions separately. Any new feature means tracking down every update location. Easy to miss one → bugs.

### With Reactivity (Vue):
All those UI elements are bound to `isLoggedIn`, `user.courses`, `user.theme` etc. When the login completes and you set `this.isLoggedIn = true`, everything updates automatically through the binding system.

> **Key Insight**: Reactivity shifts the developer's responsibility from **"update this specific element"** to **"manage state"**. The framework handles the rest.

---

## 1.3 Approaches to State & Rendering

### Approach 1: Traditional Server-Side Rendering (SSR)

- **Where state lives**: Server (session, database).
- **How rendering works**: On every state change, client sends a request → server processes it → server renders full HTML page → sends entire new HTML back → browser replaces/reloads the page.

```
User Action → HTTP Request → Server processes → Full HTML Response → Browser re-renders page
```

**Advantages**: Simple mental model, good SEO by default, single source of truth.
**Disadvantages**: Full page reloads are slow; poor UX for dynamic/interactive apps; high server load for frequent updates.

---

### Approach 2: Traditional Client-Side JavaScript (jQuery era)

- **Where state lives**: Server (for persistence) + ad-hoc JS variables on the client.
- **How rendering works**: JS code manually selects DOM elements and updates them one-by-one.

```javascript
// Login happens → manually update every affected element
$.ajax('/api/login', { success: function(user) {
  $('#nav-username').text(user.name);
  $('#nav-login-btn').hide();
  $('#nav-logout-btn').show();
  $('#course-list').html(renderCourses(user.courses));
  // ... (forgot to update notifications? → bug)
}});
```

**Advantages**: Only parts of the page update (no full reload), faster feel.
**Disadvantages**:
- Tedious and error-prone (easy to miss updating an element).
- State becomes scattered across many variables and DOM nodes.
- Code becomes tangled and hard to maintain as app grows ("spaghetti code").

---

### Approach 3: Vue's Declarative Approach ✅

- **Where state lives**: A centralized reactive `data` object on the Vue instance.
- **How rendering works**: Template bindings are declared once. Vue's reactivity system tracks which template regions depend on which data. When data changes, only those regions re-render.

```html
<template>
  <div>
    <span v-if="isLoggedIn">Welcome, {{ user.name }}</span>
    <button v-if="!isLoggedIn" @click="login">Login</button>
    <button v-if="isLoggedIn" @click="logout">Logout</button>
  </div>
</template>

<script>
export default {
  data() {
    return { isLoggedIn: false, user: null }
  }
}
</script>
```

When `isLoggedIn` flips to `true`, Vue automatically updates every binding that depends on it.

**Advantages**:
- No manual DOM manipulation.
- Single source of truth in `data`.
- Easier to reason about, test, and maintain.

---

## 1.4 Vue Directives (`v-bind`, `v-model`, `v-on`)

Directives are **special HTML attributes prefixed with `v-`** that tell Vue to do something special with a DOM element. They are the bridge between your reactive data and the HTML template.

---

### `v-bind` — One-Way Data Binding

**Direction**: Data → DOM (JavaScript → HTML attribute)

Reactively binds a JavaScript expression to an HTML attribute. When the JS value changes, the attribute updates; but the user cannot change the JS value through the DOM element.

```html
<!-- Longhand -->
<img v-bind:src="user.avatarUrl" v-bind:alt="user.name">

<!-- Shorthand (colon syntax) -->
<img :src="user.avatarUrl" :alt="user.name">

<!-- Dynamic class binding -->
<div :class="isActive ? 'active' : 'inactive'">...</div>

<!-- Boolean attribute -->
<button :disabled="isLoading">Submit</button>
```

**Use when**: You want the HTML attribute to reflect JS state, but you don't need the HTML element to write back into JS.

---

### `v-model` — Two-Way Data Binding

**Direction**: Data ↔ DOM (bidirectional synchronization)

Keeps a form input element's value perfectly in sync with a JavaScript variable in both directions:
- If JS variable changes → input value updates.
- If user types in the input → JS variable updates.

```html
<input v-model="searchQuery" placeholder="Search...">
<p>You typed: {{ searchQuery }}</p>
```

```javascript
data() {
  return {
    searchQuery: ''  // Stays perfectly in sync with <input> above
  }
}
```

**Works on**: `<input>`, `<textarea>`, `<select>`, custom components (with `modelValue` prop).

**Under the hood**, `v-model` is syntactic sugar for:
```html
<!-- v-model on an input is equivalent to: -->
<input :value="searchQuery" @input="searchQuery = $event.target.value">
```

| Control | Synced Property | Synced Event |
|---|---|---|
| `<input type="text">` | `value` | `input` |
| `<input type="checkbox">` | `checked` | `change` |
| `<select>` | `value` | `change` |

---

### `v-on` — Event Binding

**Direction**: DOM → Data (user action triggers JS function)

Listens to DOM events and executes a JavaScript method or expression when they fire.

```html
<!-- Longhand -->
<button v-on:click="handleLogin">Login</button>

<!-- Shorthand (@ syntax) -->
<button @click="handleLogin">Login</button>

<!-- Inline expression -->
<button @click="count++">Increment</button>

<!-- With event modifiers -->
<form @submit.prevent="handleSubmit">...</form>  <!-- .prevent calls event.preventDefault() -->
<input @keyup.enter="search">                    <!-- .enter fires only on Enter key -->
```

**Common Events**: `click`, `input`, `change`, `submit`, `keyup`, `keydown`, `mouseover`, `focus`, `blur`

**Event Modifiers**: `.prevent`, `.stop`, `.once`, `.capture`, `.self`, `.passive`

---

### Summary Comparison

| Directive | Alias | Direction | Best For |
|---|---|---|---|
| `v-bind` | `:` | JS → DOM | Binding JS values to HTML attributes |
| `v-model` | — | JS ↔ DOM | Form inputs, two-way sync |
| `v-on` | `@` | DOM → JS | Handling user events |

---

## 1.5 Class and Style Binding

Vue provides special handling for `:class` and `:style` bindings because manipulating an element's classes and inline styles is one of the most common UI patterns.

### `:class` — Dynamic Class Binding

#### Object Syntax (Most Common)
Pass an object where **keys are class names** and **values are booleans**. A class is added when its value is `true`, removed when `false`.

```html
<div :class="{ active: isActive, 'text-danger': hasError }">
  Content
</div>
```

```javascript
data() {
  return {
    isActive: true,    // → 'active' class IS applied
    hasError: false    // → 'text-danger' class is NOT applied
  }
}
// Result: <div class="active">Content</div>
```

You can also bind to a computed object for cleaner templates:
```javascript
computed: {
  classObject() {
    return {
      active: this.isActive && !this.hasError,
      'text-danger': this.hasError && this.errorType === 'fatal'
    }
  }
}
```
```html
<div :class="classObject">...</div>
```

#### Array Syntax
Apply a list of classes:
```html
<div :class="[activeClass, errorClass]">...</div>
<!-- Renders: <div class="active text-danger"> -->
```

#### Mixing Static and Dynamic
```html
<div class="base-styles" :class="{ active: isActive }">...</div>
<!-- Static 'base-styles' always present; 'active' is conditional -->
```

---

### `:style` — Inline Style Binding

#### Object Syntax
```html
<div :style="{ color: activeColor, fontSize: fontSize + 'px' }">...</div>
```
```javascript
data() {
  return {
    activeColor: '#42b883',  // Vue green!
    fontSize: 18
  }
}
```

#### Bind to a Style Object (Recommended for clarity):
```javascript
computed: {
  cardStyles() {
    return {
      backgroundColor: this.theme.bg,
      borderColor: this.theme.border,
      padding: '1rem'
    }
  }
}
```
```html
<div :style="cardStyles">...</div>
```

> **Note**: Vue automatically handles vendor prefixes for CSS properties that require them (e.g., `transform`, `transition`).

---

## 1.6 Conditional Rendering (`v-if` vs `v-show`)

Both control whether an element is visible, but they work very differently under the hood — and that difference matters for performance.

---

### `v-if` — True Conditional Rendering

The element is **added to or removed from the DOM** based on the condition.

```html
<div v-if="isLoggedIn">
  Welcome back, {{ user.name }}!
</div>
<div v-else>
  Please log in.
</div>
```

**How it works internally**:
- When condition is `true` → Vue **mounts** the element into the real DOM.
- When condition becomes `false` → Vue **unmounts and destroys** the element (and its children, watchers, event listeners).

**With `v-else-if` and `v-else`**:
```html
<div v-if="role === 'admin'">Admin Panel</div>
<div v-else-if="role === 'editor'">Editor Dashboard</div>
<div v-else>Read-only View</div>
```

**`<template>` with `v-if`** (group multiple elements without a wrapper):
```html
<template v-if="isLoaded">
  <h1>{{ title }}</h1>
  <p>{{ description }}</p>
</template>
```

---

### `v-show` — CSS Display Toggle

The element is **always rendered and stays in the DOM**. Only its CSS `display` property is toggled.

```html
<div v-show="isMenuOpen">
  <!-- Menu is always in DOM; just hidden/shown via CSS -->
  <ul>...</ul>
</div>
```

- When `isMenuOpen = true` → `display` is its natural value (e.g., `block`).
- When `isMenuOpen = false` → Vue inlines `style="display: none;"`.

> **Important**: `v-show` does NOT work with `<template>` tags or `v-else`.

---

### `v-if` vs `v-show` — When to Use Which?

| Feature | `v-if` | `v-show` |
|---|---|---|
| **DOM presence** | Removed when false | Always in DOM |
| **Initial render cost** | Low (if initially false, nothing is rendered) | Higher (always renders) |
| **Toggle cost** | High (mount/unmount cycle) | Very Low (just CSS toggle) |
| **Supports `v-else`** | ✅ Yes | ❌ No |
| **Lifecycle hooks fire?** | ✅ Yes (created, mounted, destroyed) | ❌ No (element stays alive) |
| **Best for** | Conditions that rarely change | Conditions that toggle frequently |

**Rule of Thumb**:
- Use `v-if` when the condition is unlikely to change at runtime (e.g., user role, feature flags).
- Use `v-show` when you need to toggle visibility frequently (e.g., dropdown menus, modals, tooltips).

---

## 1.7 Looping & Iteration (`v-for`)

`v-for` renders a list of elements by iterating over a JavaScript iterable. It's the Vue equivalent of a `for` loop in your template.

### Iterating Over Arrays

```html
<ul>
  <li v-for="course in courses" :key="course.id">
    {{ course.name }} — {{ course.instructor }}
  </li>
</ul>
```

With index:
```html
<li v-for="(course, index) in courses" :key="course.id">
  {{ index + 1 }}. {{ course.name }}
</li>
```

---

### Iterating Over Objects

```html
<!-- Just values -->
<p v-for="value in userProfile">{{ value }}</p>

<!-- Value + key (property name) -->
<p v-for="(value, key) in userProfile">
  <strong>{{ key }}:</strong> {{ value }}
</p>

<!-- Value + key + index -->
<p v-for="(value, key, index) in userProfile">
  {{ index }}. {{ key }}: {{ value }}
</p>
```

```javascript
data() {
  return {
    userProfile: {
      name: 'Rajesh Kumar',
      rollNo: 'DS21B001',
      program: 'B.Sc. Data Science'
    }
  }
}
// Renders:
// 0. name: Rajesh Kumar
// 1. rollNo: DS21B001
// 2. program: B.Sc. Data Science
```

---

### Iterating Over a Range of Numbers

```html
<!-- Renders: 1, 2, 3, 4, 5 -->
<span v-for="n in 5" :key="n">{{ n }} </span>
```

---

### `v-for` on `<template>`

To render multiple elements per iteration without adding an extra wrapper element:
```html
<template v-for="item in items" :key="item.id">
  <dt>{{ item.term }}</dt>
  <dd>{{ item.definition }}</dd>
</template>
```

---

### Template Syntax Comparison (Jinja vs Vue)

| Feature | Jinja2 (Server-side) | Vue `v-for` (Client-side) |
|---|---|---|
| Loop syntax | `{% for item in items %}` | `v-for="item in items"` |
| Interpolation | `{{ item.name }}` | `{{ item.name }}` |
| Index access | `loop.index` | `(item, index) in items` |
| End marker | `{% endfor %}` | (none needed — on the element itself) |

---

## 1.8 Loop Keys & Virtual DOM Diffing

This is one of the most important performance concepts in Vue. Understanding *why* `:key` exists requires understanding how Vue updates the DOM.

### The Virtual DOM

Vue doesn't directly manipulate the real browser DOM for every data change (that would be slow). Instead it maintains a **Virtual DOM** — a lightweight JavaScript tree representation of what the real DOM should look like.

**Process**:
1. Data changes.
2. Vue re-renders the Virtual DOM tree (fast, in-memory JS operation).
3. Vue **diffs** (compares) the new Virtual DOM with the previous Virtual DOM snapshot.
4. Vue computes the **minimal set of real DOM operations** needed.
5. Vue applies only those changes to the real browser DOM.

```
Data Change
    │
    ▼
New Virtual DOM Tree
    │
    ▼ (diff algorithm)
Old Virtual DOM Tree ─────→ Compute minimal diff
                                  │
                                  ▼
                           Patch real DOM (only changed nodes)
```

---

### Why Diffing Fails Without `:key`

Consider this list:
```javascript
items = ['Apple', 'Banana', 'Cherry']
```
```html
<li v-for="item in items">{{ item }}</li>
```

If you **prepend** an item:
```javascript
items = ['Avocado', 'Apple', 'Banana', 'Cherry']
```

Without `:key`, Vue uses a simple **positional heuristic** — it compares elements by their position in the list:
- Position 0: `Apple` → `Avocado` → **Update**
- Position 1: `Banana` → `Apple` → **Update**
- Position 2: `Cherry` → `Banana` → **Update**
- Position 3: *(new)* → `Cherry` → **Insert**

Vue re-renders **all 4 elements** even though only a single item was prepended. With complex components (having their own state, animations, inputs), this causes **incorrect state** being preserved in the wrong components.

---

### `:key` — The Solution

The `:key` attribute gives Vue a **stable identity** for each rendered element/component. Vue uses it to track which virtual node corresponds to which real DOM element across renders.

```html
<li v-for="item in items" :key="item.id">
  {{ item.name }}
</li>
```

With `:key`:
- Position 0: key=`avocado-id` (new) → **Insert**
- Position 1: key=`apple-id` → **Reuse** (move if needed)
- Position 2: key=`banana-id` → **Reuse**
- Position 3: key=`cherry-id` → **Reuse**

Vue now only **inserts one new element** and efficiently reuses the rest.

---

### `:key` Best Practices

```html
<!-- ✅ BEST: Use a unique, stable ID from your data -->
<li v-for="user in users" :key="user.id">{{ user.name }}</li>

<!-- ⚠️ ACCEPTABLE for static, non-reordering lists: Use index -->
<li v-for="(item, index) in staticList" :key="index">{{ item }}</li>

<!-- ❌ AVOID: Using index when list can be reordered/filtered -->
<!-- Using index as key on a dynamic list defeats the purpose -->
```

> **Why avoid index as key?** If the list gets sorted or filtered, items' indices change, but Vue will still try to reuse DOM elements at the same index — leading to stale component state bugs.

---

### Summary: `v-for` with `:key` — Full Example

```html
<div v-for="course in enrolledCourses" :key="course.courseId" class="course-card">
  <h3>{{ course.title }}</h3>
  <p>Instructor: {{ course.instructor }}</p>
  <span :class="{ completed: course.isCompleted }">
    {{ course.isCompleted ? '✅ Completed' : '⏳ In Progress' }}
  </span>
</div>
```

```javascript
data() {
  return {
    enrolledCourses: [
      { courseId: 'CS101', title: 'Intro to Python', instructor: 'Dr. A', isCompleted: true },
      { courseId: 'MA201', title: 'Linear Algebra', instructor: 'Dr. B', isCompleted: false },
      { courseId: 'DS301', title: 'Machine Learning', instructor: 'Dr. C', isCompleted: false },
    ]
  }
}
```

---

## Quick Reference Summary

| Concept | Directive / Feature | Key Point |
|---|---|---|
| Reactivity | (framework core) | Data changes → UI auto-updates |
| One-way bind | `v-bind` / `:` | JS → DOM attribute |
| Two-way bind | `v-model` | JS ↔ Form input (bidirectional) |
| Event handling | `v-on` / `@` | DOM event → JS method |
| Class binding | `:class="{ name: bool }"` | Toggle CSS classes via boolean object |
| Style binding | `:style="{ prop: val }"` | Inline CSS via JS object |
| DOM conditional | `v-if` / `v-else` | Mounts/unmounts DOM node |
| CSS conditional | `v-show` | Toggles `display: none` |
| List rendering | `v-for="item in list"` | Renders element per item |
| Identity tracking | `:key="unique-id"` | Efficient Virtual DOM diffing |
