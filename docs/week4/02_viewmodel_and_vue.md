# 2. Architectural Pattern: ViewModel & Vue

> **Module**: MAD II — Week 4 | **Topic**: Vue.js Fundamentals
> **Subtopics**: 2.1 → 2.7

---

## 2.1 Model vs View (The Problem of Derived / UI State)

### The Classic Split

Every application fundamentally works with two layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                            MODEL                                │
│  The "truth" of your application — persisted data               │
│  Lives in: Database, API responses, server sessions             │
│                                                                 │
│  Examples: users table, posts table, orders table               │
└──────────────────────────────┬──────────────────────────────────┘
                               │
              (data flows up, events flow down)
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                             VIEW                                │
│  What the user sees — the rendered HTML/CSS interface           │
│  Lives in: Browser DOM, templates                               │
│                                                                 │
│  Examples: login form, dashboard cards, navigation bar          │
└─────────────────────────────────────────────────────────────────┘
```

### The Gap: Where Does UI-Specific State Live?

Here's the problem: **not all state belongs in the database Model**, and **not all state is simple enough to live directly in the View template**. There is a large category of data that:

- Is derived from the model (computed/aggregated).
- Is transient and exists only during a UI session (e.g., form validation messages).
- Represents UI interaction state (e.g., "is this dropdown open?", "is this form loading?").
- Needs transforming before display (e.g., formatting a raw timestamp into "3 hours ago").

This creates a conceptual gap. Where does this *middle* data live?

### Concrete Tension

| Question | Too simple to put in DB? | Too complex for the template? |
|---|---|---|
| Is this user's password valid? | ✅ Yes, never persist | ✅ Yes, needs logic |
| How many characters remain in a textarea? | ✅ Never persist | ✅ Computed from input length |
| Is this accordion section expanded? | ✅ Session-only UI state | ✅ Needs a variable |
| What are the top 5 posts this week? | ❌ Might be derived from DB | ✅ Needs aggregation logic |

The **ViewModel** pattern was invented precisely to fill this gap.

---

## 2.2 Concrete Examples: Model vs ViewModel Data

### Example 1: Registration Form

Consider a user registration form:

```
Fields: username, email, password, repeat_password
```

**Model perspective** (what gets saved to the database):
```javascript
// users table
{
  id: 123,
  username: "rajesh_k",
  email: "rajesh@example.com",
  password_hash: "$2b$10$...",   // hashed, never plain text
  // ← repeat_password is NOT here. It doesn't belong in the DB.
}
```

**ViewModel perspective** (what the form needs to function):
```javascript
{
  // mirrors model fields (for input binding)
  username: "",
  email: "",
  password: "",

  // UI-only state — never persisted
  repeat_password: "",        // for client-side validation only
  isPasswordVisible: false,   // toggle show/hide password
  isSubmitting: false,        // show loading spinner during API call
  validationErrors: {},       // form error messages per field
  isEmailAvailable: null,     // result of async email-check API call
}
```

`repeat_password` has zero business being in the database — it's a **UI concern**. The ViewModel is the right home for it.

---

### Example 2: Blog Dashboard

A dashboard shows `top_posts` and `top_comments` for the week.

**Model perspective** (database tables):
```
posts:    id, title, author_id, body, created_at, view_count, like_count
comments: id, post_id, author_id, body, created_at, like_count
```

**The design question**: Should `top_posts` be a separate DB table, or computed on-the-fly?

| Option | Approach | Trade-off |
|---|---|---|
| **Separate table** | Pre-computed, stored in DB | Fast to query, but stale; complex invalidation logic |
| **Computed ViewModel property** | Derived at request-time from `posts` sorted by `view_count` | Always accurate, slightly more compute |

`top_posts` is often better modelled as a **derived/computed ViewModel property** (sorted/filtered slice of the posts model), not a standalone database entity.

---

### The Guiding Principle

> **Ask yourself**: "Would I ever write a migration to add this to the database, or does it disappear when the user closes the browser tab?"
>
> - If it disappears → **ViewModel state**
> - If it must persist → **Model state**

---

## 2.3 The ViewModel Pattern & Data Binding

### What is a ViewModel?

The **ViewModel** is an architectural construct that sits between the Model and the View. It:

1. **Holds presentation state** — data that only exists for display purposes.
2. **Holds derived state** — data computed or transformed from the raw Model.
3. **Exposes methods** — actions the View can trigger (form submit, toggle state).
4. **Binds to the View** — via a data-binding mechanism so changes propagate automatically.

```
┌─────────────┐        ┌────────────────────────────┐        ┌─────────────┐
│             │        │         VIEWMODEL           │        │             │
│    MODEL    │◄──────►│  - Presentation state       │◄──────►│    VIEW     │
│  (database) │  sync  │  - Derived/computed data    │ binds  │  (template) │
│             │        │  - UI interaction state     │        │             │
└─────────────┘        │  - Methods/actions          │        └─────────────┘
                       └────────────────────────────┘
```

### How Data Binding Works

**One-Way Binding** (ViewModel → View):
- ViewModel property changes → View automatically re-renders.
- The View is a "projection" of the ViewModel's current state.

**Two-Way Binding** (ViewModel ↔ View):
- ViewModel property changes → View updates.
- User edits View input (types in a field) → ViewModel property updates.
- This is what `v-model` achieves in Vue.

```
ViewModel: { searchQuery: "" }
     ↕ (v-model)
View: <input type="text">

User types "Vue.js"
     ↓
ViewModel: { searchQuery: "Vue.js" }  ← automatically updated
     ↓
All parts of the view that depend on `searchQuery` re-render instantly
```

### Primary Benefits

| Benefit | Explanation |
|---|---|
| **Separation of concerns** | Business data (Model) stays clean; UI logic lives in ViewModel |
| **Testability** | ViewModel logic can be unit tested without a browser |
| **Reusability** | Multiple Views can share the same ViewModel |
| **Maintainability** | Changes to UI logic don't touch the database model |
| **Declarative templates** | Views become simple, readable HTML with data references |

---

## 2.4 Vue as a ViewModel / Vue Instance

### Vue's Relationship with MVVM

Vue.js was explicitly inspired by the MVVM (Model-View-ViewModel) pattern. The Vue documentation even states:

> *"Vue is inspired by MVVM though it doesn't strictly follow the MVVM pattern."*

The key inspiration is the **data-binding mechanism** — the `data` object of a Vue instance functions as the ViewModel.

### Anatomy of a Vue Instance as a ViewModel

```javascript
const app = Vue.createApp({

  // ─── VIEWMODEL STATE ───────────────────────────────────────────────
  data() {
    return {
      // Raw data (mirrors model fields)
      username: '',
      password: '',

      // UI-only state (pure ViewModel concerns)
      isPasswordVisible: false,
      isLoading: false,
      errorMessage: '',
    }
  },

  // ─── DERIVED STATE (Computed ViewModel Properties) ──────────────────
  computed: {
    isFormValid() {
      return this.username.length >= 3 && this.password.length >= 8;
    },
    passwordStrengthLabel() {
      if (this.password.length < 6)  return 'Weak';
      if (this.password.length < 10) return 'Medium';
      return 'Strong';
    }
  },

  // ─── VIEWMODEL ACTIONS ──────────────────────────────────────────────
  methods: {
    async handleLogin() {
      this.isLoading = true;
      try {
        await api.login(this.username, this.password);
      } catch (e) {
        this.errorMessage = e.message;
      } finally {
        this.isLoading = false;
      }
    },
    togglePasswordVisibility() {
      this.isPasswordVisible = !this.isPasswordVisible;
    }
  }
});
```

**Corresponding View (template)**:
```html
<form @submit.prevent="handleLogin">

  <input v-model="username" placeholder="Username" />

  <input
    v-model="password"
    :type="isPasswordVisible ? 'text' : 'password'"
    placeholder="Password"
  />
  <button type="button" @click="togglePasswordVisibility">
    {{ isPasswordVisible ? 'Hide' : 'Show' }}
  </button>

  <span :class="'strength-' + passwordStrengthLabel.toLowerCase()">
    Password strength: {{ passwordStrengthLabel }}
  </span>

  <p v-if="errorMessage" class="error">{{ errorMessage }}</p>

  <button type="submit" :disabled="!isFormValid || isLoading">
    {{ isLoading ? 'Logging in...' : 'Login' }}
  </button>

</form>
```

The **template never contains logic** — it's a pure reflection of the ViewModel state. All logic lives in `data`, `computed`, and `methods`.

### The Bi-Directional Synchronization Cycle

```
User types in <input>
       ↓
v-model detects input event
       ↓
Updates ViewModel data (e.g., username = "rajesh")
       ↓
Vue's reactivity system detects the change
       ↓
Re-evaluates computed properties that depend on username
       ↓
Re-renders template regions that use username or those computed props
       ↓
User sees updated UI immediately (password strength label, button state, etc.)
```

---

## 2.5 MVC vs MVVM Architecture

### MVC — Model-View-Controller

The classic web application pattern (Flask, Django, Rails all use this):

```
┌──────────┐  User Input   ┌────────────┐   Reads/Writes   ┌─────────┐
│          │──────────────►│            │◄────────────────►│         │
│   VIEW   │               │ CONTROLLER │                  │  MODEL  │
│          │◄──────────────│            │                  │         │
└──────────┘  Renders View └────────────┘                  └─────────┘
```

**Roles**:
- **Model**: Data + business logic. Talks to the database.
- **View**: Renders the UI. Usually a template (Jinja, HTML).
- **Controller**: The "glue". Handles incoming requests, talks to the Model, selects and renders the View.

**In Flask terms**:
```python
# Controller (route handler)
@app.route('/dashboard')
def dashboard():
    user = User.query.get(session['user_id'])   # talks to Model
    posts = Post.query.filter_by(author=user).all()
    return render_template('dashboard.html',    # selects & renders View
                           user=user, posts=posts)
```

---

### MVVM — Model-View-ViewModel

A refinement primarily for **rich client-side UIs**:

```
┌──────────┐  Data Binding  ┌────────────────┐   Data Access   ┌─────────┐
│          │◄──────────────►│                │◄───────────────►│         │
│   VIEW   │                │   VIEWMODEL    │                 │  MODEL  │
│(Template)│  Commands/     │  (Vue instance)│                 │  (API/  │
│          │──────────────► │                │                 │   DB)   │
└──────────┘  Events        └────────────────┘                 └─────────┘
```

**Roles**:
- **Model**: Same as MVC — data from API/database.
- **View**: The HTML template. Now **passive** — it only expresses bindings, no logic.
- **ViewModel**: The new layer. Exposes data and commands to the View. Manages UI state.

---

### Are They Mutually Exclusive? No.

A full-stack app typically uses **both MVC on the server** and **MVVM on the client**:

```
Browser (Client)                          Server
─────────────────────────────────         ──────────────────────────
Vue Template (View)                       Flask Route (Controller)
      ↕ (data binding)                          ↕
Vue Instance (ViewModel)   ←── HTTP ──►   Flask Model (SQLAlchemy)
      ↕ (API calls)                             ↕
   REST API layer                          PostgreSQL / SQLite DB
```

The **Controller** (Flask route) handles:
- Authentication/authorization.
- Routing the request to the right handler.
- Fetching data from the Model.
- Returning JSON for the Vue ViewModel to consume.

The **ViewModel** (Vue instance) handles:
- Managing UI state (loading, errors, form values).
- Derived/computed data for the template.
- Calling the API (triggering controller actions).

> **Key Insight**: MVVM doesn't replace MVC — it *refines the View layer* of MVC into View + ViewModel. They operate at different layers of the stack.

---

### Side-by-Side Comparison

| Aspect | MVC | MVVM |
|---|---|---|
| **Primary domain** | Server-side web apps | Client-side rich UIs |
| **View role** | Active (renders logic) | Passive (declarative bindings only) |
| **Who handles UI state** | Controller (partially) | ViewModel (dedicated) |
| **Binding mechanism** | Manual (template renders once) | Automatic (reactive data binding) |
| **Testing UI logic** | Harder (tied to request/response) | Easier (ViewModel is pure JS) |
| **Examples** | Flask, Django, Rails, Laravel | Vue, Angular, Knockout.js |

---

## 2.6 Computed Properties

### The Problem They Solve

Computed properties are Vue's dedicated abstraction for **derived data** — values that are calculated based on other reactive state.

**Without computed properties** (wrong way — logic polluting the template):
```html
<!-- ❌ Messy, hard to read, impossible to test, not cached -->
<p>Full Name: {{ firstName.trim() + ' ' + lastName.trim() }}</p>
<p>Status: {{ score >= 90 ? 'Distinction' : score >= 75 ? 'First Class' : 'Pass' }}</p>
<ul>
  <li v-for="link in (isAdmin ? adminLinks : userLinks)">{{ link.label }}</li>
</ul>
```

**With computed properties** (correct way):
```html
<!-- ✅ Template is clean and readable -->
<p>Full Name: {{ fullName }}</p>
<p>Status: {{ gradeStatus }}</p>
<ul>
  <li v-for="link in navLinks">{{ link.label }}</li>
</ul>
```

```javascript
computed: {
  fullName() {
    return `${this.firstName.trim()} ${this.lastName.trim()}`;
  },
  gradeStatus() {
    if (this.score >= 90) return 'Distinction';
    if (this.score >= 75) return 'First Class';
    return 'Pass';
  },
  navLinks() {
    return this.isAdmin ? this.adminLinks : this.userLinks;
  }
}
```

---

### Key Property 1: Auto-Updating (Reactive Dependencies)

Computed properties **automatically track their reactive dependencies**. When any dependency changes, the computed property re-evaluates and the View updates.

```javascript
data() {
  return {
    items: [
      { name: 'Vue.js', price: 0, inStock: true },
      { name: 'React',  price: 0, inStock: false },
      { name: 'Svelte', price: 0, inStock: true },
    ],
    showOnlyInStock: false,
  }
},
computed: {
  filteredItems() {
    // Vue tracks: this.items AND this.showOnlyInStock
    if (this.showOnlyInStock) {
      return this.items.filter(item => item.inStock);
    }
    return this.items;
  }
}
```

When `showOnlyInStock` is toggled (via a checkbox), `filteredItems` instantly re-evaluates and the list re-renders.

---

### Key Property 2: Reactive Caching (Performance)

This is the **critical difference** between computed properties and methods.

```javascript
// ──────────────────────────────────────
// As a METHOD — called every time it's referenced in the template
// ──────────────────────────────────────
methods: {
  expensiveCalculation() {
    console.log("Running expensive calculation...");
    // Imagine this loops through 10,000 items
    return this.items.reduce((sum, item) => sum + item.value, 0);
  }
}

// ──────────────────────────────────────
// As a COMPUTED — cached until dependencies change
// ──────────────────────────────────────
computed: {
  expensiveCalculation() {
    console.log("Running expensive calculation...");
    return this.items.reduce((sum, item) => sum + item.value, 0);
  }
}
```

```html
<!-- If called 3 times in template: -->
<p>{{ expensiveCalculation }}</p>   <!-- computed: runs ONCE, cached -->
<p>{{ expensiveCalculation }}</p>   <!-- computed: returns cache, no re-run -->
<p>{{ expensiveCalculation }}</p>   <!-- computed: returns cache, no re-run -->

<!-- As a method: -->
<p>{{ expensiveCalculation() }}</p>  <!-- method: runs each time -->
<p>{{ expensiveCalculation() }}</p>  <!-- method: runs again -->
<p>{{ expensiveCalculation() }}</p>  <!-- method: runs again — 3x total! -->
```

> **Cache invalidation**: The cache is only thrown away when a **reactive dependency** of the computed property changes. If `this.items` doesn't change, the cached result is returned instantly on every access.

**What about `Date.now()`?**
```javascript
computed: {
  now() {
    return Date.now();  // ⚠️ This will NEVER update!
                        // Date.now() is not a reactive dependency.
                        // Cache never invalidates. Use a method instead.
  }
}
```

---

### Key Property 3: Computed Setters

By default, computed properties are **getter-only** (read-only). But you can define a **setter** too:

```javascript
computed: {
  fullName: {
    // Getter — called when you READ this.fullName
    get() {
      return `${this.firstName} ${this.lastName}`;
    },
    // Setter — called when you SET this.fullName = "..."
    set(newValue) {
      const parts = newValue.trim().split(' ');
      this.firstName = parts[0];
      this.lastName = parts.slice(1).join(' ');
    }
  }
}
```

```javascript
// Usage:
this.fullName = 'Rajesh Kumar';
// → this.firstName becomes 'Rajesh'
// → this.lastName becomes 'Kumar'
// → getter re-evaluates → this.fullName = 'Rajesh Kumar' ✅
```

---

### Common Real-World Computed Examples

```javascript
computed: {
  // 1. Filtering a list
  activeUsers() {
    return this.users.filter(u => u.isActive);
  },

  // 2. Sorting
  sortedPosts() {
    return [...this.posts].sort((a, b) =>
      new Date(b.createdAt) - new Date(a.createdAt)
    );
  },

  // 3. Aggregation
  totalCartPrice() {
    return this.cartItems.reduce((sum, item) =>
      sum + item.price * item.quantity, 0
    );
  },

  // 4. Dynamic CSS class object
  buttonClasses() {
    return {
      'btn-primary': !this.hasError && !this.isLoading,
      'btn-danger':   this.hasError,
      'btn-disabled': this.isLoading,
    };
  },

  // 5. Dynamic nav links based on role
  navigationLinks() {
    const base = [{ label: 'Home', href: '/' }, { label: 'Profile', href: '/profile' }];
    if (this.isAdmin) base.push({ label: 'Admin', href: '/admin' });
    return base;
  }
}
```

---

## 2.7 Watchers (and Computed vs Watchers)

### What are Watchers?

Watchers are a way to **explicitly observe a specific reactive property** and run arbitrary code whenever it changes.

```javascript
watch: {
  // Watch 'searchQuery' — runs every time searchQuery changes
  searchQuery(newValue, oldValue) {
    console.log(`Search changed from "${oldValue}" to "${newValue}"`);
    this.fetchSearchResults(newValue);  // API call!
  }
}
```

The watcher function receives:
- **`newValue`**: The value the property just changed **to**.
- **`oldValue`**: The value the property was **before** the change.

---

### When to Use Watchers

Watchers are designed for **side effects** — actions that happen *as a consequence of* state change, rather than *deriving new state* from it.

**Good use cases for watchers**:

```javascript
watch: {
  // 1. Async operations (API calls on state change)
  searchQuery: debounce(async function(query) {
    if (query.length < 2) return;
    this.results = await api.search(query);
  }, 300),

  // 2. Watching a route parameter
  '$route.params.id'(newId) {
    this.loadCourseDetails(newId);
  },

  // 3. Persisting state to localStorage on change
  userPreferences: {
    handler(newPrefs) {
      localStorage.setItem('prefs', JSON.stringify(newPrefs));
    },
    deep: true   // ← watch nested object changes too
  },

  // 4. Triggering animations or timers
  isModalOpen(isOpen) {
    if (isOpen) this.startFocusTrap();
    else this.releaseFocusTrap();
  }
}
```

---

### Watcher Options

```javascript
watch: {
  someProperty: {
    handler(newVal, oldVal) { /* ... */ },

    // immediate: true → runs the handler immediately on component creation
    //            (not just when the value changes)
    immediate: true,

    // deep: true → watches nested changes inside objects/arrays
    //              (normally Vue only detects reassignment, not mutation)
    deep: true,
  }
}
```

**Example — deep watch**:
```javascript
data() {
  return {
    user: { name: 'Rajesh', address: { city: 'Chennai' } }
  }
},
watch: {
  user: {
    handler(newUser) {
      console.log("User object changed:", newUser);
    },
    deep: true   // Without this, changing user.address.city would NOT trigger the watcher
  }
}
```

---

### Computed vs Watchers — The Decision Table

This is a common source of confusion. Here is when to use each:

| Criterion | Use `computed` | Use `watch` |
|---|---|---|
| **Purpose** | Derive a new value from state | Perform a side effect when state changes |
| **Return value** | Always returns something | Usually returns nothing |
| **Async support** | ❌ Must be synchronous | ✅ Can be async |
| **Side effects (API, DOM, timer)** | ❌ Avoid | ✅ Designed for this |
| **Caching** | ✅ Automatic (reactive) | ❌ No caching, runs every change |
| **Dependencies tracked automatically?** | ✅ Yes | ❌ Must explicitly name the property |
| **Common examples** | Filtered lists, totals, formatted strings | API calls, localStorage sync, animations |

**The Golden Rule**:
> ✅ If you can express it as "give me X, derived from Y" → **`computed`**
> ✅ If you need to "do something *when* Y changes" → **`watch`**

**Anti-pattern to avoid**:
```javascript
// ❌ Don't use a watcher to derive/compute state
watch: {
  firstName(val) {
    this.fullName = val + ' ' + this.lastName;  // Setting another data property
  }
}

// ✅ Use a computed instead
computed: {
  fullName() {
    return this.firstName + ' ' + this.lastName;
  }
}
```

---

## Quick Reference Summary

| Concept | Vue Feature | Key Point |
|---|---|---|
| **UI-only state** | `data()` | Transient, session-based, never persisted |
| **Derived/presentation state** | `computed` | Auto-updates, cached, synchronous |
| **Side effects on state change** | `watch` | Async-capable, explicit, for API/DOM ops |
| **ViewModel binding** | `v-model`, `v-bind` | View auto-reflects ViewModel state |
| **Controller role** | Server route / Vuex action | Orchestrates data flow between Model & ViewModel |
| **MVC + MVVM** | Both co-exist | Server uses MVC; client View layer uses MVVM |

### The Full MVVM Flow in Vue

```
Database (Model)
     ↕ (HTTP/REST)
Flask Route (Controller) ──► returns JSON
     ↓
Vue instance data() (ViewModel)
     ↕ (reactive binding)
Vue template (View)
     ↑ (user events via @click, v-model)
User Interaction
```
