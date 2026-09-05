# Module 2: Client-Side Routing & Vue Router

---

## 1. Page Composition — Server Pages vs. Component Trees

### 1.1 How Traditional Servers Served Pages (Multi-Page Applications)

In classic server-rendered web development (think plain HTML sites, old Flask/Django with Jinja templates, PHP), the application is a **collection of separate HTML documents** living on the server. Every distinct URL maps to a distinct HTML file or a server-generated HTML response.

**The classic request-response cycle:**

```
User types: https://example.com/about
        │
        ▼
Browser sends:   GET /about HTTP/1.1
        │
        ▼
Server receives request, queries DB if needed,
renders Jinja/PHP template → full HTML string
        │
        ▼
Browser receives full HTML (200 OK)
Discards current page entirely
Parses HTML from scratch
Downloads CSS, JS, images again
        │
        ▼
New page fully renders (~500ms – 3s)
```

**What gets lost every single navigation:**
- Scroll position on the previous page
- Partially filled form inputs
- In-memory JavaScript state (cart, user preferences, open modals)
- Any ongoing audio/video playback

**What gets re-downloaded wastefully on every page:**
- The same `<header>`, `<nav>`, and `<footer>` HTML
- The same `app.css` and `app.js` files
- Fonts, logo images, common icons

### 1.2 The Modern Component Tree Model

Vue applications are not a sequence of disconnected pages. They are a **single, persistent JavaScript application** structured as a tree of nested components:

```
                    App.vue  (always alive, never destroyed)
                   /        \
          <nav>               <main>
       AppNavbar.vue        <router-view>
                           /            \
                    Home.vue    OR    Profile.vue   (only ONE is active at a time)
                   /      \
            HeroSection  FeaturedList
```

- The **shell** (navbar, footer, layout) stays mounted indefinitely.
- Only the **inner view component** — whatever is inside `<router-view>` — is swapped when the URL changes.
- No full-page reload. No white flash. No wasted re-downloads.

---

## 2. What is Vue Router?

**Vue Router** is the **official routing library** for Vue.js, maintained by the Vue core team. It provides deep integration with Vue's reactivity system so URL changes automatically drive component rendering.

### Core responsibilities:
1. **Map URL paths to components** — when the URL is `/user/5`, render `UserProfile.vue`
2. **Manage browser navigation history** — back/forward buttons work correctly
3. **Intercept anchor clicks** — prevent default browser GET requests
4. **Expose route metadata** reactively to every component

---

## 3. Vue Router Setup & Core Directives

### 3.1 The Two Essential Template Elements

#### `<router-link>`

```html
<router-link to="/about">About Us</router-link>
```

Renders as a standard `<a>` tag in the DOM, but **intercepts the click event** before it can trigger a browser navigation request. Instead, it tells Vue Router to update the URL and swap the active component.

**Why not just use `<a href="/about">`?**
A plain `<a>` tag sends a real HTTP GET request to the server, causing a full page reload — which defeats the entire purpose of an SPA.

| Feature | `<a href="...">` | `<router-link to="...">` |
| :--- | :--- | :--- |
| Triggers page reload | ✅ Yes | ❌ No |
| Works with Vue Router history | ❌ No | ✅ Yes |
| Adds `router-link-active` class | ❌ No | ✅ Yes (automatic) |
| Supports named routes / params | ❌ No | ✅ Yes |

**`router-link-exact-active`**: Vue Router automatically adds this CSS class to the `<router-link>` whose path **exactly** matches the current URL. Useful for highlighting the active nav item.

```css
/* Style the currently active nav link */
nav a.router-link-exact-active {
  color: #42b983;
  font-weight: bold;
  border-bottom: 2px solid #42b983;
}
```

#### `<router-view>`

```html
<router-view></router-view>
```

This is the **dynamic outlet** — a placeholder in the template that gets replaced with whichever component matches the current URL. When the route changes, Vue Router unmounts the old component from this slot and mounts the new one.

There is **nothing to configure** on `<router-view>` itself for basic usage — it just sits in your template and Vue Router handles the rest.

### 3.2 Complete Step-by-Step Setup

Setting up Vue Router from scratch requires four steps:

```javascript
// ── Step 1: Import Vue and VueRouter ──────────────────────────
import Vue from 'vue'
import VueRouter from 'vue-router'

// ── Step 2: Register VueRouter as a Vue plugin ────────────────
// This injects $router and $route into every Vue component globally
Vue.use(VueRouter)

// ── Step 3: Define route components ───────────────────────────
const Home    = { template: '<div><h2>Home Page</h2></div>' }
const About   = { template: '<div><h2>About Page</h2></div>' }
const Contact = { template: '<div><h2>Contact Page</h2></div>' }

// ── Step 4: Define the routes table ───────────────────────────
// Each route maps a URL path to a component
const routes = [
  { path: '/',        component: Home    },
  { path: '/about',   component: About   },
  { path: '/contact', component: Contact },
  { path: '*',        redirect: '/'      } // Catch-all: redirect unknown paths to home
]

// ── Step 5: Instantiate VueRouter with the routes ─────────────
const router = new VueRouter({
  routes // shorthand for routes: routes
})

// ── Step 6: Inject router into the root Vue instance ──────────
// This makes this.$router and this.$route available in all child components
const app = new Vue({
  router,
  template: `
    <div id="app">
      <nav>
        <router-link to="/">Home</router-link>
        <router-link to="/about">About</router-link>
        <router-link to="/contact">Contact</router-link>
      </nav>
      <hr>
      <router-view></router-view>
    </div>
  `
}).$mount('#app')
```

### 3.3 Two Key Injected Properties

Once the router is injected into the root instance, **every component** (no matter how deeply nested) has automatic access to:

| Property | Type | Purpose | Example Usage |
| :--- | :--- | :--- | :--- |
| `this.$router` | Router instance | Programmatically navigate | `this.$router.push('/about')` |
| `this.$route` | Route object (reactive) | Read current URL metadata | `this.$route.params.id` |

**`this.$router` — Navigation Methods:**
```javascript
this.$router.push('/about')               // Navigate to /about (adds to history)
this.$router.push({ name: 'user', params: { id: 42 } }) // Named route navigation
this.$router.replace('/login')            // Navigate without adding to history
this.$router.go(-1)                       // Go back one step (like browser back button)
this.$router.go(1)                        // Go forward one step
```

**`this.$route` — Current Route Data:**
```javascript
this.$route.path      // '/user/42'
this.$route.params    // { id: '42' }
this.$route.query     // { tab: 'posts', page: '2' } (from /user/42?tab=posts&page=2)
this.$route.hash      // '#section-3'
this.$route.name      // 'user-profile' (if route has a name)
this.$route.fullPath  // '/user/42?tab=posts#section-3'
```

---

## 4. Advantages of Client-Side Routing

### 4.1 Zero Full-Page Reloads

```
MPA Navigation:          SPA Navigation (Vue Router):
  ~800ms - 3000ms              ~0ms - 50ms
  ┌───────────┐                ┌───────────┐
  │  request  │                │  JS runs  │
  │  latency  │   vs           │  in RAM   │
  │  parse    │                │  patches  │
  │  render   │                │  DOM only │
  └───────────┘                └───────────┘
```

Navigation feels **instantaneous** because no network request is made for HTML — the component is already in JavaScript memory (or is lazily loaded as a small chunk).

### 4.2 Preserved Application State

Because the shell application is never destroyed, all state persists through navigation:
- Vuex store data stays intact
- In-progress form drafts survive
- Media players keep playing
- WebSocket connections remain open
- Scroll restoration can be precisely controlled

### 4.3 Reduced Network Traffic

Instead of fetching full HTML pages, the app fetches **only the data it needs** as JSON:

```
MPA navigation to /products:
  ← 45 KB full HTML (header + nav + footer + product list + scripts)

SPA navigation to /products:
  ← 1.2 KB JSON  { "products": [...] }
    Component already in memory, only data changes
```

### 4.4 Smooth Transitions & Animations

Since Vue controls exactly when old and new components mount/unmount, it is trivial to add animated transitions between views using Vue's built-in `<transition>` wrapper:

```html
<transition name="fade" mode="out-in">
  <router-view></router-view>
</transition>
```

```css
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s ease; }
.fade-enter, .fade-leave-to { opacity: 0; }
```

---

## 5. Dynamic Route Matching & Params

### 5.1 The Problem Dynamic Routes Solve

A social network has millions of users. You don't create a separate route for each one:
```
❌ Wrong:
/user/alice   → AliceProfileComponent
/user/bob     → BobProfileComponent
/user/charlie → CharlieProfileComponent
```

Instead, you define **one route with a dynamic segment** that captures the variable part:

```javascript
✅ Correct:
{ path: '/user/:username', component: UserProfile }
// Matches: /user/alice, /user/bob, /user/charlie, /user/any-name-at-all
```

### 5.2 Defining Dynamic Segments — Colon Syntax

A colon (`:`) before a path segment name marks it as a **dynamic parameter**:

```javascript
const routes = [
  { path: '/user/:id',                       component: UserProfile },
  { path: '/post/:year/:month/:slug',         component: BlogPost    },
  { path: '/product/:category/:productId',    component: ProductPage },
]
```

### 5.3 Accessing Params Inside Components

When a dynamic route is matched, Vue Router populates `this.$route.params` with an object containing each named segment's value:

```html
<!-- UserProfile.vue -->
<template>
  <div>
    <h2>User Profile</h2>
    <p>Showing profile for user ID: <strong>{{ $route.params.id }}</strong></p>
    <button @click="loadUser">Load Data</button>
  </div>
</template>

<script>
export default {
  name: 'UserProfile',
  created() {
    this.loadUser();
  },
  methods: {
    loadUser() {
      const userId = this.$route.params.id;
      // Use userId to fetch from API
      fetch(`/api/users/${userId}`)
        .then(res => res.json())
        .then(data => { this.user = data; });
    }
  },
  data() {
    return { user: null };
  }
};
</script>
```

### 5.4 Multi-Segment Param Table

| Route Pattern | Matched URL | `$route.params` result |
| :--- | :--- | :--- |
| `/user/:id` | `/user/42` | `{ id: '42' }` |
| `/user/:id` | `/user/john` | `{ id: 'john' }` |
| `/post/:year/:month` | `/post/2024/09` | `{ year: '2024', month: '09' }` |
| `/files/:pathMatch(.*)` | `/files/img/logo.png` | `{ pathMatch: 'img/logo.png' }` |

> [!NOTE]
> All `$route.params` values are **strings**, even if they look like numbers. Convert explicitly: `Number(this.$route.params.id)` or `parseInt(this.$route.params.id)`.

---

## 6. Component Reuse & Reactivity Caveats

### 6.1 Vue's Component Reuse Optimization

When you navigate from `/user/alice` to `/user/bob`, both routes match the **same component** (`UserProfile`). Vue Router takes an optimization shortcut:

> **Instead of destroying `UserProfile` and creating a fresh one, Vue reuses the existing instance.**

This is efficient — teardown and creation of a component involves DOM manipulation, garbage collection of event listeners, and re-initialization. For identical components, this is unnecessary overhead.

### 6.2 The Critical Problem

Because the component is **reused** (not recreated), its lifecycle hooks **do not fire again**:

```javascript
export default {
  name: 'UserProfile',
  created() {
    // ⚠️  This runs ONCE when first mounting at /user/alice
    // ⚠️  It does NOT run again when navigating to /user/bob
    this.fetchUser(this.$route.params.id);
  }
}
```

**Result**: Navigating from `/user/alice` to `/user/bob` leaves `alice`'s profile data on screen even though the URL shows `/user/bob`. The component thinks it's still alice's page.

### 6.3 Solution 1 — Watch `$route`

React to parameter changes by placing a watcher on `$route`:

```javascript
export default {
  name: 'UserProfile',
  data() {
    return { user: null };
  },
  created() {
    // Runs on initial mount
    this.fetchUser(this.$route.params.id);
  },
  watch: {
    // Runs every time the route object changes (including param changes)
    '$route'(to, from) {
      if (to.params.id !== from.params.id) {
        this.fetchUser(to.params.id);
      }
    }
    // Alternatively, watch only the specific param:
    // '$route.params.id'(newId, oldId) {
    //   this.fetchUser(newId);
    // }
  },
  methods: {
    fetchUser(id) {
      fetch(`/api/users/${id}`)
        .then(res => res.json())
        .then(data => { this.user = data; });
    }
  }
};
```

### 6.4 Solution 2 — In-Component Navigation Guard (`beforeRouteUpdate`)

Vue Router provides a dedicated lifecycle hook that fires specifically when the component's route updates while it is being reused:

```javascript
export default {
  name: 'UserProfile',
  data() {
    return { user: null };
  },
  created() {
    this.fetchUser(this.$route.params.id);
  },
  // Fires when navigating to the same component with new params
  beforeRouteUpdate(to, from, next) {
    this.fetchUser(to.params.id); // Use `to.params` not `this.$route.params`
    next(); // MUST call next() to confirm the navigation
  },
  methods: {
    fetchUser(id) {
      fetch(`/api/users/${id}`)
        .then(res => res.json())
        .then(data => { this.user = data; });
    }
  }
};
```

### 6.5 Solution 3 — Force Recreation with `:key`

If you explicitly want Vue to destroy and recreate the component on every param change, bind a `:key` attribute to `<router-view>`:

```html
<!-- In App.vue or parent template -->

<!-- WITHOUT :key — component is reused (default) -->
<router-view></router-view>

<!-- WITH :key — component is destroyed and recreated on every route change -->
<router-view :key="$route.fullPath"></router-view>
```

**Trade-off**: `:key` forces recreation which is simpler to reason about (lifecycle hooks always fire), but it discards component state and can be slightly slower than reuse for heavy components.

---

## 7. Advanced Vue Router Capabilities

### 7.1 Nested Routes

Real UIs have nested layouts. A user settings page might have sub-sections: Account, Privacy, Notifications — each with its own URL but all sharing the same settings sidebar layout.

```
/settings/account       → SettingsLayout + AccountTab
/settings/privacy       → SettingsLayout + PrivacyTab
/settings/notifications → SettingsLayout + NotificationsTab
```

**Configuration using `children`:**

```javascript
const routes = [
  {
    path: '/settings',
    component: SettingsLayout,   // Parent: renders the sidebar + nested <router-view>
    children: [
      // Empty path → default child rendered at exactly /settings
      { path: '',          component: AccountTab        },
      { path: 'account',   component: AccountTab        },
      { path: 'privacy',   component: PrivacyTab        },
      { path: 'notifications', component: NotificationsTab }
    ]
  }
]
```

**Parent component must include its own `<router-view>`:**

```html
<!-- SettingsLayout.vue -->
<template>
  <div class="settings-page">
    <aside class="settings-sidebar">
      <router-link to="/settings/account">Account</router-link>
      <router-link to="/settings/privacy">Privacy</router-link>
      <router-link to="/settings/notifications">Notifications</router-link>
    </aside>
    <main class="settings-content">
      <!-- Child route component renders HERE -->
      <router-view></router-view>
    </main>
  </div>
</template>
```

### 7.2 Named Routes

Instead of hardcoding path strings throughout your codebase, give routes a `name`:

```javascript
const routes = [
  { path: '/user/:id',  name: 'user-profile',  component: UserProfile },
  { path: '/post/:slug', name: 'blog-post',    component: BlogPost    },
  { path: '/login',      name: 'login',         component: Login       }
]
```

**Why named routes?**

```javascript
// ❌ Fragile: hardcoded string paths everywhere
<router-link to="/user/42">Profile</router-link>
this.$router.push('/user/' + this.userId)

// ✅ Robust: if the path changes from /user/:id to /profile/:id,
//    you only update the routes table — nothing else breaks
<router-link :to="{ name: 'user-profile', params: { id: 42 } }">Profile</router-link>
this.$router.push({ name: 'user-profile', params: { id: this.userId } })
```

Named routes also support query strings and hash:
```javascript
this.$router.push({
  name: 'blog-post',
  params: { slug: 'vue-router-guide' },
  query: { ref: 'newsletter' },
  hash: '#comments'
})
// Navigates to: /post/vue-router-guide?ref=newsletter#comments
```

### 7.3 Named Views (Multiple Simultaneous Viewports)

Sometimes a single route needs to render **multiple independent components side by side**, not nested inside each other. For example, a dashboard layout with a dedicated header area, a sidebar, and a main content area — all driven by the same route.

```html
<!-- App.vue template with multiple named router-view outlets -->
<template>
  <div id="app">
    <router-view name="topbar"></router-view>      <!-- Named: topbar -->
    <div class="layout">
      <router-view name="sidebar"></router-view>   <!-- Named: sidebar -->
      <router-view></router-view>                  <!-- Unnamed: default -->
    </div>
  </div>
</template>
```

```javascript
const routes = [
  {
    path: '/dashboard',
    // Note: "components" (PLURAL) when using named views
    components: {
      default:  DashboardMain,    // → renders into unnamed <router-view>
      topbar:   DashboardTopbar,  // → renders into <router-view name="topbar">
      sidebar:  DashboardSidebar  // → renders into <router-view name="sidebar">
    }
  },
  {
    path: '/settings',
    components: {
      default:  SettingsMain,
      topbar:   SettingsTopbar,  // Different topbar for settings section
      sidebar:  SettingsSidebar
    }
  }
]
```

> [!IMPORTANT]
> The property is **`components`** (plural) when using named views, versus **`component`** (singular) for a single view. Mixing them up is a very common mistake.

### 7.4 HTML5 History Mode vs. Hash Mode

Vue Router supports two URL modes. The choice affects how URLs look and whether you need server configuration.

#### Hash Mode (Default)

```
URL:  https://example.com/#/user/42
      └──────────────────────────────── everything after # is the hash
```

- The `#` and everything after it is called the **fragment identifier** or hash.
- Browsers **never send the hash to the server** in HTTP requests. Only the part before `#` is transmitted.
- Therefore, the server always receives `GET /` regardless of the route — it works with any static file server with zero configuration.

```javascript
// Hash mode is the default — no mode property needed
const router = new VueRouter({ routes })
// Equivalent to:
const router = new VueRouter({ mode: 'hash', routes })
```

#### HTML5 History Mode

```
URL:  https://example.com/user/42
      └──────────────────────────── clean URL, no hash
```

- Uses the HTML5 `history.pushState()` API to update the URL without triggering a browser reload.
- **Produces clean, professional-looking URLs** identical to traditional server-rendered pages.
- **Requires server configuration** to avoid 404 errors.

```javascript
const router = new VueRouter({
  mode: 'history',
  routes
})
```

#### The History Mode Server Fallback Problem

When a user **directly types** `https://example.com/user/42` into the address bar or **refreshes** on that page, the browser sends a real HTTP request to the server for `GET /user/42`. If the server has no file or route handler for `/user/42`, it returns **404 Not Found**.

**Fix**: Configure the server to serve `index.html` for any path that doesn't match a static file:

```nginx
# Nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

```apache
# Apache (.htaccess)
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

```python
# Flask catch-all fallback
@app.route('/', defaults={'path': ''})
@app.route('/<path:path>')
def catch_all(path):
    return render_template('index.html')
```

Once `index.html` is served, Vue boots up and Vue Router reads the URL (`/user/42`) and renders the correct component — the user never notices anything wrong.

---

## 8. Why Client-Side Routing Leads to Single Page Applications

Client-side routing is the structural backbone that makes SPAs possible:

```
Without Vue Router (Traditional):          With Vue Router (SPA):
                                           
Server owns navigation                     JavaScript owns navigation
Every URL = server HTTP response           Every URL = component render decision
HTML generated on server per request       HTML generated in-memory by Vue
Server tightly coupled to presentation     Server is pure data API (JSON)
                                           
Results in:                                Results in:
  Multi-Page Application (MPA)               Single Page Application (SPA)
  Multiple HTML documents                    One HTML shell, many component views
  Full reloads on navigation                 Zero reloads, instant transitions
  Server knows about every "page"            Server knows nothing about "pages"
```

When you hand navigation entirely to JavaScript:
1. The **backend server is freed** from HTML rendering and routing concerns.
2. The server becomes a **pure API** — stateless, focused exclusively on data and business logic.
3. The **entire frontend** (routing, state, rendering) becomes an independent, deployable application.
4. The frontend can be served from a **CDN globally** while the API runs on dedicated servers.

---

## Summary — Key Exam Points

| Concept | Key Fact |
| :--- | :--- |
| `<router-link>` | Renders as `<a>` but intercepts click to prevent page reload |
| `<router-view>` | Dynamic outlet; replaced with matched component |
| `this.$router` | Router instance for programmatic navigation (`push`, `replace`, `go`) |
| `this.$route` | Reactive current route data (`params`, `query`, `path`, `name`) |
| Dynamic segments | Colon prefix: `/user/:id` — captured in `$route.params.id` |
| Component reuse | Vue reuses instances for same component on param change — `created()`/`mounted()` do NOT re-run |
| Fix reuse issue | Watch `'$route'`, use `beforeRouteUpdate`, or bind `:key="$route.fullPath"` on `<router-view>` |
| Nested routes | `children: [...]` in route config; parent must contain its own `<router-view>` |
| Named routes | `name: '...'` in config; `:to="{ name: '...', params: { ... } }"` in template |
| Named views | `components: { default, name1, name2 }` (plural!); multiple `<router-view name="...">` outlets |
| Hash mode | Default; `/#/path`; no server config needed; hash never sent to server |
| History mode | `mode: 'history'`; clean URLs; **requires server fallback** to `index.html` to avoid 404s |
