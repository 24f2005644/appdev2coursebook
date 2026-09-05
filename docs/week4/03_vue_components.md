# 3. Vue Components

> **Module**: MAD II — Week 4 | **Topic**: Vue.js Fundamentals
> **Subtopics**: 3.1 → 3.4

---

## 3.1 Motivations: Code Reuse & Refactoring (DRY Principle)

### The Problem: Repetition in UI

Imagine building a course listing page for the IITM portal. Without components, you'd write something like this:

```html
<!-- Without components — copy-pasting the same HTML block repeatedly -->

<div class="course-card">
  <img src="/courses/python.jpg" alt="Python">
  <h3>Introduction to Python</h3>
  <p>Dr. Arpit Gupta · 12 weeks</p>
  <span class="tag">Beginner</span>
  <button>Enroll</button>
</div>

<div class="course-card">
  <img src="/courses/ml.jpg" alt="Machine Learning">
  <h3>Machine Learning Foundations</h3>
  <p>Dr. Priya Rajan · 16 weeks</p>
  <span class="tag">Intermediate</span>
  <button>Enroll</button>
</div>

<div class="course-card">
  <img src="/courses/dbms.jpg" alt="DBMS">
  <h3>Database Management</h3>
  <p>Dr. Srinivasan · 10 weeks</p>
  <span class="tag">Beginner</span>
  <button>Enroll</button>
</div>

<!-- ... 20 more courses? Copy-paste 20 more times? -->
```

**Problems with this approach**:
- If you want to add a "Wishlist" button to every card → edit every single copy.
- If a CSS class name changes → hunt down every occurrence.
- The HTML file balloons to thousands of lines for a real catalogue.
- One typo → one broken card (hard to spot).

---

### Motivation 1: Reuse (DRY — Don't Repeat Yourself)

> **DRY Principle**: Every piece of knowledge (logic, structure, style) must have a **single, unambiguous, authoritative representation** within a system.

**With a component**:
```html
<!-- CourseCard.vue — defined ONCE -->
<template>
  <div class="course-card">
    <img :src="course.thumbnail" :alt="course.title">
    <h3>{{ course.title }}</h3>
    <p>{{ course.instructor }} · {{ course.duration }} weeks</p>
    <span class="tag">{{ course.level }}</span>
    <button @click="$emit('enroll', course.id)">Enroll</button>
  </div>
</template>
```

```html
<!-- Parent page — clean and expressive -->
<CourseCard v-for="course in courses" :key="course.id" :course="course" @enroll="handleEnroll" />
```

Now adding a "Wishlist" button means editing **one file** — `CourseCard.vue`. All 20+ card instances update automatically.

**Real-world component reuse patterns**:

| Pattern | Example |
|---|---|
| **Card components** | IITM portal course cards, Amazon product tiles, Twitter posts |
| **Navigation items** | Sidebar links, breadcrumbs, tab bars |
| **Form controls** | Custom `<DatePicker>`, `<RatingStars>`, `<SearchInput>` |
| **Layout wrappers** | `<PageLayout>`, `<Modal>`, `<Accordion>` |
| **Notification/badges** | `<Toast>`, `<Badge count="5">`, `<Alert type="error">` |

---

### Motivation 2: Refactoring

**Refactoring** means restructuring code to improve its internal quality **without changing external behaviour**. Components enable this at scale.

**Before refactoring** (monolithic single-file page):
```
index.html — 1,500 lines
├── Navigation HTML (lines 1–80)
├── Hero banner HTML (lines 81–140)
├── Course listing HTML (lines 141–900)  ← massive and unwieldy
├── Footer HTML (lines 901–980)
└── <script> with 400 lines of Vue logic
```

**After refactoring** (componentised):
```
App.vue
├── <NavBar />         →  components/NavBar.vue       (80 lines)
├── <HeroBanner />     →  components/HeroBanner.vue   (60 lines)
├── <CourseList />     →  components/CourseList.vue   (50 lines)
│     └── <CourseCard /> → components/CourseCard.vue (40 lines)
└── <Footer />         →  components/Footer.vue       (80 lines)
```

**Benefits of refactoring into components**:
- **Isolation**: A bug in `CourseCard` cannot break `NavBar`.
- **Parallel development**: Team members can work on different components simultaneously without conflicts.
- **Readability**: `App.vue` becomes a high-level description of the page structure.
- **Testability**: Each component can be unit-tested in isolation.
- **Replaceability**: Swap out `NavBar.vue` entirely without touching anything else.

---

## 3.2 Vue Component Structure (Props, Data, Template)

### The Three Pillars of a Component

A Vue component is a **self-contained, reusable unit** that bundles together:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Vue Component                            │
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │   PROPS     │    │      DATA        │    │   TEMPLATE    │  │
│  │             │    │                  │    │               │  │
│  │  Inputs     │    │  Private state   │    │  HTML markup  │  │
│  │  from       │───►│  + computed      │───►│  rendered to  │  │
│  │  parent     │    │  + methods       │    │  the DOM      │  │
│  │             │    │  + watchers      │    │               │  │
│  └─────────────┘    └──────────────────┘    └───────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Props — The Component's Public Interface

**Props** (short for *properties*) are the mechanism by which a **parent component passes data down to a child component**.

Think of props as **function parameters** — they let the parent customise how each instance of a component renders and behaves.

```javascript
// CourseCard.vue — Child component
export default {
  name: 'CourseCard',

  props: {
    // Simple type declaration
    title: String,

    // With validation rules
    instructor: {
      type: String,
      required: true,
    },
    duration: {
      type: Number,
      default: 12,
    },
    level: {
      type: String,
      default: 'Beginner',
      validator(value) {
        return ['Beginner', 'Intermediate', 'Advanced'].includes(value);
      }
    },
    thumbnailUrl: {
      type: String,
      required: true,
    }
  }
}
```

**Parent passing props**:
```html
<!-- Static prop value -->
<CourseCard title="Intro to Python" instructor="Dr. Arpit" :duration="12" />

<!-- Dynamic prop (from parent's data) -->
<CourseCard
  v-for="course in courses"
  :key="course.id"
  :title="course.title"
  :instructor="course.instructor"
  :duration="course.duration"
  :level="course.level"
  :thumbnailUrl="course.thumbnail"
/>
```

**Critical rule — Props are read-only in the child**:
```javascript
// ❌ NEVER mutate a prop directly in the child
this.title = "New Title";  // Vue will warn about this

// ✅ If you need to modify it locally, copy it into local data
data() {
  return {
    localTitle: this.title  // copy prop → local data
  }
}
```

**Prop Types Supported**:
`String`, `Number`, `Boolean`, `Array`, `Object`, `Date`, `Function`, `Symbol`

---

### Data — Private, Encapsulated State

The `data()` function returns the **component instance's private reactive state**. This state is:
- **Isolated**: Not accessible from outside the component (unless emitted or returned via an exposed ref).
- **Reactive**: Changes trigger the component's template to re-render.
- **Instance-scoped**: Each component instance gets its own separate copy of `data`.

```javascript
// CourseCard.vue
export default {
  props: ['course'],

  data() {
    return {
      // Private UI state — not passed in from outside
      isWishlisted: false,
      isEnrolling: false,
      showDetails: false,
    }
  },

  computed: {
    wishlistIcon() {
      return this.isWishlisted ? '❤️' : '🤍';
    }
  },

  methods: {
    toggleWishlist() {
      this.isWishlisted = !this.isWishlisted;
    },
    async enroll() {
      this.isEnrolling = true;
      await api.enroll(this.course.id);
      this.isEnrolling = false;
      this.$emit('enrolled', this.course.id);
    }
  }
}
```

**Why `data` must be a function (not a plain object)**:

```javascript
// ❌ WRONG — plain object (shared between ALL instances)
data: {
  isWishlisted: false   // all 20 cards share the same object!
}

// ✅ CORRECT — factory function (each instance gets its own copy)
data() {
  return {
    isWishlisted: false   // each card has its own independent copy
  }
}
```

If `data` were a plain object, toggling `isWishlisted` on one card would toggle it on **every** card simultaneously — because they'd all reference the same object in memory.

---

### Template — The Component's HTML Blueprint

The `template` defines how the component's data and props are **rendered to HTML**. It is the View layer of the component's mini-MVVM.

```html
<!-- CourseCard.vue — Full SFC (Single File Component) -->
<template>
  <div class="course-card" :class="{ 'is-wishlisted': isWishlisted }">

    <img :src="course.thumbnailUrl" :alt="course.title" class="thumbnail" />

    <div class="card-body">
      <h3 class="course-title">{{ course.title }}</h3>
      <p class="meta">{{ course.instructor }} · {{ course.duration }} weeks</p>
      <span class="level-badge" :class="'level-' + course.level.toLowerCase()">
        {{ course.level }}
      </span>
    </div>

    <div class="card-actions">
      <button class="wishlist-btn" @click="toggleWishlist">
        {{ wishlistIcon }}
      </button>
      <button
        class="enroll-btn"
        :disabled="isEnrolling"
        @click="enroll"
      >
        {{ isEnrolling ? 'Enrolling...' : 'Enroll Now' }}
      </button>
    </div>

  </div>
</template>
```

---

### Single File Components (SFC) — The `.vue` File

Vue's recommended way to write components puts template, script, and style all in **one `.vue` file**:

```
CourseCard.vue
├── <template>  ← HTML structure
├── <script>    ← JavaScript logic (props, data, methods, computed)
└── <style>     ← CSS scoped to this component only
```

```html
<template>
  <div class="card">{{ course.title }}</div>
</template>

<script>
export default {
  name: 'CourseCard',
  props: { course: Object },
  data() {
    return { isWishlisted: false }
  }
}
</script>

<style scoped>
/* 'scoped' means these styles ONLY apply to this component */
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
}
</style>
```

**`<style scoped>`**: Vue generates a unique attribute (e.g., `data-v-3ba67c28`) and rewrites CSS selectors to `.card[data-v-3ba67c28]` — so styles don't leak out and clash with other components.

---

### Component Registration

Before using a component, it must be **registered**:

```javascript
// ─── Global Registration (available everywhere in the app) ───
const app = Vue.createApp({});
app.component('CourseCard', CourseCard);   // registered globally

// ─── Local Registration (only available in this component) ───
import CourseCard from './CourseCard.vue';

export default {
  components: {
    CourseCard   // shorthand for CourseCard: CourseCard
  }
}
```

**When to use which**:
- **Global**: Truly universal UI primitives (e.g., `<BaseButton>`, `<BaseInput>`).
- **Local**: Feature-specific components. Preferred — better for tree-shaking (unused components aren't bundled).

---

### Component Communication Patterns

Components communicate through a well-defined system:

```
        Parent
          │
          │  Props (data flows DOWN ↓)
          ▼
        Child
          │
          │  $emit (events flow UP ↑)
          ▼
        Parent

```

**Child emitting events to parent**:
```javascript
// Child: CourseCard.vue
methods: {
  enroll() {
    this.$emit('enroll', this.course.id);  // emit event with payload
  }
}
```

```html
<!-- Parent: CoursePage.vue -->
<CourseCard
  :course="course"
  @enroll="handleEnroll"    <!-- listen for the emitted event -->
/>
```

```javascript
// Parent
methods: {
  handleEnroll(courseId) {
    console.log(`Enrolling in course: ${courseId}`);
  }
}
```

---

## 3.3 Templates & Render Functions

### Template Syntax

Vue templates are **valid HTML** extended with special syntax. The Vue compiler compiles them into efficient JavaScript render functions at build time.

#### Text Interpolation — Mustache `{{ }}`

```html
<h1>Hello, {{ user.name }}!</h1>
<p>Score: {{ score * 10 }}%</p>
<span>{{ isLoggedIn ? 'Online' : 'Offline' }}</span>
```

The content inside `{{ }}` is a **JavaScript expression** evaluated in the context of the component instance. Can contain: variable references, arithmetic, ternary operators, method calls.

**Cannot contain**: `if` statements, `for` loops, variable declarations (`let`, `const`). These belong in `computed` or `methods`.

```html
<!-- ❌ NOT valid inside {{ }} -->
{{ if (score > 90) { return 'A'; } }}

<!-- ✅ Use ternary instead -->
{{ score > 90 ? 'A' : 'B' }}
```

---

#### Jinja vs Vue Template — Syntactic Similarity

| Feature | Jinja2 (Python) | Vue (JavaScript) |
|---|---|---|
| Interpolation | `{{ variable }}` | `{{ variable }}` |
| Conditional | `{% if condition %}` | `v-if="condition"` |
| Loop | `{% for item in list %}` | `v-for="item in list"` |
| Attribute binding | `href="{{ url }}"` | `:href="url"` |
| Filters/pipes | `{{ name\|upper }}` | `{{ name.toUpperCase() }}` |
| Template inheritance | `{% extends 'base.html' %}` | Components + Slots |

The syntax was deliberately made familiar to developers coming from Jinja/Mustache server-side templating.

---

#### Built-In Security: Auto-Escaping (XSS Prevention)

Vue **automatically HTML-escapes** all content rendered via `{{ }}`.

```javascript
data() {
  return {
    // Imagine this came from user input / a database
    userInput: '<script>alert("XSS Attack!")</script>'
  }
}
```

```html
<p>{{ userInput }}</p>
<!-- Renders as literal text: <script>alert("XSS Attack!")</script>  -->
<!-- The <script> tag is NOT executed — it's escaped to:             -->
<!-- &lt;script&gt;alert("XSS Attack!")&lt;/script&gt;              -->
```

**What if you actually need to render HTML** (from a trusted source)?
```html
<!-- v-html bypasses escaping — ONLY use with trusted content! -->
<div v-html="trustedHtmlContent"></div>
```
> ⚠️ **Never use `v-html` on user-generated content.** It opens the door to XSS vulnerabilities.

---

#### Syntax Enforcement

The Vue template compiler also performs **structural validation** at compile time:
- Detects unclosed tags: `<div><p>content</div>` → compile error.
- Warns on invalid nesting: `<p><div>...</div></p>` → invalid HTML structure.
- Type checks props (if using TypeScript or prop validators).
- Catches undefined variables in development mode.

This turns what would be silent runtime bugs in plain HTML into **loud compile-time errors**.

---

### Render Functions — Programmatic Rendering

Vue templates are syntactic sugar that compile down to **render functions**. In most cases, you'll never need to write render functions manually. But understanding them explains how Vue works internally.

**What a template compiles to**:
```html
<!-- Template -->
<div class="card">
  <h3>{{ title }}</h3>
  <p v-if="isVisible">{{ description }}</p>
</div>
```

```javascript
// Compiled render function (what Vue actually executes)
import { createElementVNode as _c, openBlock as _ob, createBlock as _cb,
         createCommentVNode as _cv, toDisplayString as _s } from 'vue'

render() {
  return _c('div', { class: 'card' }, [
    _c('h3', null, _s(this.title)),
    this.isVisible
      ? _c('p', null, _s(this.description))
      : _cv("v-if", true)
  ])
}
```

**When to write render functions manually**:

You'd use render functions directly when the logic to determine *what* to render is so dynamic that a declarative template becomes awkward:

```javascript
// Example: A component that renders different heading levels (h1-h6)
// based on a prop — this is clunky as a template but clean as a render fn
import { h } from 'vue';

export default {
  props: ['level', 'text'],   // level: 1, 2, 3, 4, 5, or 6

  render() {
    return h(`h${this.level}`, this.text);
    // Renders: <h1>text</h1> or <h2>text</h2>, etc.
  }
}
```

---

### JSX — HTML + JavaScript Combined

Vue also supports **JSX** (JavaScript XML) — a syntax extension that lets you write HTML-like markup directly inside JavaScript. JSX is the default syntax in React and an option in Vue.

```jsx
// ─── JSX in Vue (similar to React) ───────────────────────────

import { defineComponent } from 'vue';

export default defineComponent({
  props: ['course'],

  setup(props) {
    const handleEnroll = () => {
      console.log('Enrolling in:', props.course.title);
    };

    // JSX — return directly from render function
    return () => (
      <div class="course-card">
        <h3>{props.course.title}</h3>
        <p>{props.course.instructor}</p>
        <button onClick={handleEnroll}>Enroll</button>
      </div>
    );
  }
});
```

**JSX vs Vue Template**:

| | Vue Template (`.vue`) | JSX |
|---|---|---|
| **Syntax** | HTML + directives (`v-if`, `v-for`) | JavaScript + inline HTML |
| **Familiarity** | Familiar to HTML/Django/Jinja devs | Familiar to React devs |
| **Dynamic rendering** | `v-if`, `v-for` directives | Native JS (`if`, `map()`) |
| **Compile step needed?** | Yes (via Vue compiler) | Yes (via Babel/JSX transform) |
| **When to use** | Almost always (preferred) | Complex dynamic rendering, render function replacement |

---

## 3.4 Component Slots

### What are Slots?

Slots are **content distribution outlets** — they let a parent component inject its own template content into a designated location inside a child component's template.

Think of a slot like a **hole** or **placeholder** in the component's template that the parent can **fill in** with custom content.

**Without slots**, components can only be customised via props — and props can only pass data (strings, numbers, objects), not arbitrary HTML structure.

**With slots**, the parent can pass in anything: text, HTML, even other components.

---

### Basic / Default Slot

```html
<!-- BaseCard.vue — defines a slot -->
<template>
  <div class="base-card">
    <div class="card-header">
      <h2>{{ title }}</h2>
    </div>
    <div class="card-body">
      <slot></slot>   <!-- 👈 Parent content goes HERE -->
    </div>
  </div>
</template>

<script>
export default {
  props: ['title']
}
</script>
```

```html
<!-- Parent — fills the slot with custom content -->
<BaseCard title="Course Overview">
  <!-- Everything between the tags goes into <slot> -->
  <p>This course covers Python fundamentals.</p>
  <ul>
    <li>Variables & Data Types</li>
    <li>Control Flow</li>
    <li>Functions</li>
  </ul>
  <button>Start Learning</button>
</BaseCard>
```

**Rendered output**:
```html
<div class="base-card">
  <div class="card-header">
    <h2>Course Overview</h2>
  </div>
  <div class="card-body">
    <p>This course covers Python fundamentals.</p>
    <ul>
      <li>Variables & Data Types</li>
      <li>Control Flow</li>
      <li>Functions</li>
    </ul>
    <button>Start Learning</button>
  </div>
</div>
```

---

### Fallback / Default Slot Content

You can provide **fallback content** that renders when the parent doesn't fill the slot:

```html
<!-- BaseCard.vue -->
<slot>
  <!-- Fallback content — shown if parent provides nothing -->
  <p class="empty-state">No content provided.</p>
</slot>
```

```html
<!-- Parent provides content → fallback is ignored -->
<BaseCard title="My Card">
  <p>Custom content here!</p>
</BaseCard>

<!-- Parent provides nothing → fallback renders -->
<BaseCard title="Empty Card" />
```

---

### Named Slots — Multiple Slots in One Component

A component can have **multiple slots**, each with a unique name:

```html
<!-- AppLayout.vue — a page layout component -->
<template>
  <div class="layout">

    <header class="layout-header">
      <slot name="header">
        <h1>Default Header</h1>   <!-- fallback -->
      </slot>
    </header>

    <main class="layout-body">
      <slot></slot>   <!-- default (unnamed) slot -->
    </main>

    <aside class="layout-sidebar">
      <slot name="sidebar"></slot>
    </aside>

    <footer class="layout-footer">
      <slot name="footer">
        <p>© 2026 IITM</p>       <!-- fallback -->
      </slot>
    </footer>

  </div>
</template>
```

**Parent filling named slots using `v-slot`**:
```html
<AppLayout>
  <!-- Fill the 'header' slot -->
  <template v-slot:header>
    <h1>Dashboard</h1>
    <nav>...</nav>
  </template>

  <!-- Fill the default slot (no name) -->
  <CourseList :courses="courses" />

  <!-- Fill the 'sidebar' slot — shorthand: #sidebar -->
  <template #sidebar>
    <RecommendedCourses />
    <ProgressTracker />
  </template>

  <!-- 'footer' slot not provided → fallback "© 2026 IITM" renders -->
</AppLayout>
```

**Shorthand**: `v-slot:header` can be written as `#header`.

---

### Scoped Slots — Child Passes Data Back to Parent

Scoped slots are the most powerful slot variant. They allow the **child component to pass data back up to the parent** within the slot — letting the parent decide how to render that data.

**Use case**: A `CourseList` component fetches and manages the courses internally, but lets the parent decide how each course should be rendered (card, row, grid, etc.).

```html
<!-- CourseList.vue — child exposes each course via scoped slot -->
<template>
  <div class="course-list">
    <div v-for="course in courses" :key="course.id">
      <!-- Pass 'course' data up to whoever fills this slot -->
      <slot :course="course" :index="index"></slot>
    </div>
  </div>
</template>
```

```html
<!-- Parent — receives the 'course' data and decides how to render it -->

<!-- Option A: Render as cards -->
<CourseList>
  <template #default="{ course }">
    <CourseCard :course="course" />
  </template>
</CourseList>

<!-- Option B: Render as a simple list row -->
<CourseList>
  <template #default="{ course, index }">
    <div class="list-row">
      {{ index + 1 }}. {{ course.title }} — {{ course.instructor }}
    </div>
  </template>
</CourseList>
```

The same `CourseList` component works with **both** layouts without any changes to the component itself. The parent controls the rendering strategy.

---

### Slots vs Props — When to Use Which?

| Scenario | Use Props | Use Slots |
|---|---|---|
| Passing plain data (text, numbers, booleans) | ✅ | ❌ |
| Passing an object or array | ✅ | ❌ |
| Passing HTML structure | ❌ (use `v-html` carefully) | ✅ |
| Passing other components | ❌ | ✅ |
| Letting parent control layout/structure | ❌ | ✅ |
| Sharing child's internal data with parent's template | ❌ | ✅ Scoped Slot |

---

## Quick Reference Summary

| Concept | Feature | Key Point |
|---|---|---|
| **Why components?** | DRY Principle | Define once, reuse everywhere; isolate changes |
| **Parent → Child data** | `props` | Read-only in child; typed and validated |
| **Private component state** | `data()` (function) | Must be a factory function for per-instance isolation |
| **Component rendering** | `<template>` | HTML + Vue directives; compiled to render functions |
| **Text output** | `{{ expression }}` | Auto-escaped, prevents XSS |
| **Raw HTML output** | `v-html` | Bypasses escaping — trusted content only |
| **Programmatic rendering** | Render functions / JSX | For highly dynamic structures |
| **Default content injection** | `<slot>` | Parent fills placeholder in child template |
| **Multiple injection points** | Named slots (`v-slot:name` / `#name`) | Multiple holes in one component |
| **Child data to parent template** | Scoped slots (`:course="course"`) | Child exposes data; parent decides rendering |
| **Child → Parent communication** | `$emit('event', payload)` | Events flow up; props flow down |

### Component Architecture Pattern

```
┌─────────────── App.vue ──────────────────┐
│                                          │
│  <NavBar />           <Footer />         │
│                                          │
│  <CourseList>                            │
│    <template #default="{ course }">      │
│      <CourseCard                         │
│        :course="course"          ← Props │
│        @enroll="handleEnroll"   ← Events │
│      >                                   │
│        <template #actions>      ← Slots  │
│          <WishlistButton />              │
│        </template>                       │
│      </CourseCard>                       │
│    </template>                           │
│  </CourseList>                           │
│                                          │
└──────────────────────────────────────────┘
```
