# 3. Managing Components & Single File Components (SFC)

---

## 3.1 Limitations of Traditional / In-browser Components — *The Need for SFCs*

Before Single File Components existed, Vue components were defined entirely in plain JavaScript files or inline in the browser. This approach has several serious problems at scale:

---

### Problem 1: Global Namespace Collisions

When registering components globally (e.g., `Vue.component('my-button', {...})`), all component names share a **single global namespace**. In large apps:
- Two developers might independently name components the same thing.
- Third-party libraries might clash with your own component names.
- There is **no module-level isolation** — everything is globally visible.

```js
// Traditional global registration — name collisions are likely in large projects
Vue.component('Button', { template: '<button>Click</button>' });
// Somewhere else in the codebase...
Vue.component('Button', { template: '<button>Submit</button>' }); // ⚠️ Silently overwritten!
```

---

### Problem 2: Multi-line String Templates & No IDE Support

Templates were written as **JavaScript strings**, which is awkward and error-prone:

```js
Vue.component('my-card', {
  template: `
    <div class="card">
      <h2>{{ title }}</h2>
      <p>{{ body }}</p>
    </div>
  `
  // Problems:
  // - No syntax highlighting (it's just a string to the IDE)
  // - No auto-complete for HTML attributes or Vue directives
  // - No lint errors for malformed HTML
  // - Harder to read and maintain as templates grow
});
```

In contrast, `.vue` files give editors full context to provide:
- HTML syntax highlighting inside `<template>`
- Vue directive auto-complete (`v-model`, `v-for`, etc.)
- CSS colour previews inside `<style>`

---

### Problem 3: CSS Encapsulation Issues (Global CSS Leakage)

In traditional setups, all CSS is **global by default**. A style rule in one component can accidentally affect elements in a completely different component:

```html
<!-- component-a.js -->
<style> h2 { color: red; } </style>  <!-- This affects ALL h2 tags on the page! -->

<!-- component-b.js -->
<!-- The h2 here will also turn red — unintended side effect -->
```

There is no native mechanism to **scope CSS to a specific component** in a plain JS setup. SFCs solve this with the `scoped` attribute.

---

### Problem 4: No Build Step — Missing Modern Tooling

Without a build pipeline, you lose access to:

| Feature | Why it matters |
|---|---|
| **ES Modules (`import`/`export`)** | Proper dependency management; tree-shaking |
| **Babel / TypeScript** | Write modern JS or TS; compile down for older browsers |
| **CSS Preprocessors** | Sass/SCSS/Less for variables, nesting, mixins |
| **Linting & Formatting** | ESLint, Prettier integration |
| **Hot Module Replacement (HMR)** | Instantly see changes in the browser without full reload |
| **Optimized Production Builds** | Minification, code splitting, tree-shaking |

In-browser Vue (loaded via CDN `<script>` tag) is fine for prototypes, but **not viable for production-grade applications**.

---

## 3.2 SFC Architecture & Structure — `.vue` Files

A **Single File Component** is a custom file format (`.vue`) that encapsulates a component's template, logic, and styles into **one cohesive file**. It is NOT valid JavaScript or HTML on its own — it must be processed by a build tool (Vite, Webpack) before the browser can use it.

### Anatomy of a `.vue` File

```vue
<!-- MyComponent.vue -->

<!-- ① TEMPLATE BLOCK: Declarative HTML markup -->
<template>
  <div class="card">
    <h2>{{ title }}</h2>
    <button @click="greet">Say Hello</button>
  </div>
</template>

<!-- ② SCRIPT BLOCK: Component logic -->
<script>
export default {
  name: 'MyComponent',  // Component name (used for debugging, DevTools)
  props: ['title'],     // Props received from parent
  data() {
    return {
      message: 'Hello!'
    };
  },
  methods: {
    greet() {
      alert(this.message);
    }
  }
}
</script>

<!-- ③ STYLE BLOCK: Scoped CSS styling -->
<style scoped>
.card {
  border: 1px solid #ccc;
  padding: 1rem;
  border-radius: 8px;
}

h2 {
  /* 'scoped' means this ONLY affects h2 elements inside THIS component */
  color: steelblue;
}
</style>
```

### The Three Blocks in Detail

#### `<template>` — Declarative Markup
- Contains the **HTML structure** of the component.
- Must have exactly **one root element** (in Vue 2; Vue 3 allows multiple).
- Supports all Vue directives: `v-model`, `v-for`, `v-if`, `@click`, `:bind`, etc.
- The template is compiled into a **render function** by the build tool.

#### `<script>` — Component Logic
- A standard ES Module — uses `export default { ... }` to export the component options object.
- Contains: `data()`, `methods`, `computed`, `props`, `watch`, lifecycle hooks, `components`, etc.
- Can use `import` to bring in other modules, utilities, or child components.

```js
// Inside <script> — importing a child component
import ChildCard from './ChildCard.vue';

export default {
  components: { ChildCard }, // Locally registered — no namespace collision!
  data() { ... },
  methods: { ... }
}
```

#### `<style scoped>` — Scoped CSS
- `scoped` is the key attribute — it tells the Vue compiler to **add a unique data attribute** to every element in the template, and to suffix all CSS selectors with that attribute.
- Result: styles are **guaranteed to only affect elements in this component**.

```html
<!-- What you write -->
<style scoped>
h2 { color: red; }
</style>

<!-- What the compiler generates (conceptually) -->
<style>
h2[data-v-f3f3eg9] { color: red; }
</style>
<!-- And adds data-v-f3f3eg9 to every <h2> in this component's DOM -->
```

Without `scoped`, the style is still component-local in source but **global in effect** once injected into the page.

---

## 3.3 Re-evaluating Separation of Concerns

### Traditional Separation (File-Type-Based)
The classic web development philosophy: **one file per technology type**.

```
project/
├── index.html    ← Structure (HTML)
├── styles.css    ← Presentation (CSS)
└── app.js        ← Behaviour (JavaScript)
```

- ✅ Clean separation of *file types*.
- ❌ A single **logical feature** (e.g., a "Login Form") is **scattered across 3 files**.
- ❌ As the app grows, files become massive and harder to navigate.
- ❌ Changing one component's appearance requires jumping between files.

### Component-Based Cohesion (SFC Approach)
SFCs flip the philosophy: organize by **logical concern (feature/component)**, not by file type.

```
src/components/
├── LoginForm.vue     ← template + logic + styles for Login Form
├── NavBar.vue        ← template + logic + styles for Navigation
└── UserCard.vue      ← template + logic + styles for User Card
```

Everything that belongs to the **LoginForm** lives together in `LoginForm.vue`. You only ever need to open one file to understand, change, or debug a component.

### The Key Insight
> *"Separation of concerns does not mean separation of file types."*

True separation of concerns means separating **independent units of functionality** from each other — and a component *is* that independent unit. Template, logic, and styles for a button are **tightly coupled by nature**; separating them into different files doesn't decouple them, it just makes them harder to find.

| Traditional Model | SFC / Component Model |
|---|---|
| Organized by file type | Organized by feature/component |
| One concern spans many files | One component = one file |
| Hard to move/delete a feature | Easy — just delete the `.vue` file |
| Global CSS by default | Scoped CSS per component |

---

## 3.4 Modern Frontend Tooling & CLI Ecosystem

### Why a Build Step is Necessary

`.vue` files are **not valid HTML or JS** — browsers can't parse them directly. A **build tool** must:
1. Parse the `.vue` file into its three blocks.
2. Compile the `<template>` into a JavaScript render function.
3. Process the `<script>` (Babel/TypeScript transforms, module resolution).
4. Extract and scope the `<style>` (inject into the DOM or output as a CSS file).
5. Bundle everything into files the browser understands (`.js`, `.css`).

### Build Tools & Bundlers

| Tool | Description | Speed |
|---|---|---|
| **Webpack** | The original, powerful, highly configurable bundler. Used widely but config-heavy. | Moderate |
| **Vite** | Modern build tool by Vue's creator (Evan You). Uses native ES Modules in dev for near-instant startup. Esbuild-powered. | ⚡ Very Fast |
| **ESBuild** | Ultra-fast JS bundler written in Go. Often used as the transform engine inside Vite. | ⚡⚡ Fastest |

**Vite** is the current recommended tool for new Vue projects.

### Package Management with `npm`

`npm` (Node Package Manager) manages your project's **dependencies** — third-party libraries your app relies on.

```
project/
├── package.json        ← Lists all dependencies and scripts
├── package-lock.json   ← Exact locked versions (for reproducible installs)
├── node_modules/       ← Downloaded packages (never commit this!)
├── src/
│   └── components/
└── vite.config.js      ← Build tool configuration
```

**Key `npm` commands:**

```bash
npm install             # Install all dependencies listed in package.json
npm install axios       # Add a new dependency (saves to package.json)
npm run dev             # Start the development server (with HMR)
npm run build           # Build optimised production bundle
npm run preview         # Preview the production build locally
```

### CLI Workflow — Scaffolding a Vue Project

The **Vue CLI** or **`create-vue`** scaffolds a complete project structure instantly:

```bash
# Using Vite + Vue (recommended modern approach)
npm create vue@latest my-project
cd my-project
npm install
npm run dev
```

This generates a fully configured project with:
- Vite as the build tool
- Vue Router (optional)
- Pinia for state management (optional)
- ESLint + Prettier
- Vitest for unit testing

### Development vs. Production Modes

| Mode | Command | Behaviour |
|---|---|---|
| **Development** | `npm run dev` | Fast rebuilds, source maps, HMR, readable output, Vue DevTools enabled |
| **Production** | `npm run build` | Minified, tree-shaken, optimised bundle — ready to deploy to a server |

---

## Summary

```
The Case for SFCs
├── Problems with traditional components
│   ├── Global name collisions       → Solved by local component registration in SFC <script>
│   ├── String templates / no IDE    → Solved by dedicated <template> block (IDE-aware)
│   ├── Global CSS leakage           → Solved by <style scoped>
│   └── No build step                → SFCs require (and embrace) a build pipeline
│
├── SFC Structure (.vue file)
│   ├── <template>   → HTML + Vue directives (compiles to render function)
│   ├── <script>     → ES Module logic (data, methods, computed, imports)
│   └── <style scoped> → CSS scoped to this component only
│
├── Separation of Concerns
│   └── Organize by FEATURE (component), not by file type
│
└── Tooling Ecosystem
    ├── Build tools:  Vite (recommended) > Webpack > ESBuild
    ├── Package mgr:  npm (package.json, node_modules)
    └── CLI scaffold: npm create vue@latest
```
