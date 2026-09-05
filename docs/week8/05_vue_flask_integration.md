# Topic 5: How to Integrate Vue with Flask

---

## 5.1 Architectural Integration Patterns

There are two fundamentally different ways to use Vue with a Flask backend. Choosing the right pattern depends on the complexity of your project, your team's skill set, and how much interactivity you need.

---

### Pattern A: Embedded / Monolithic Approach

In this pattern, Flask remains the primary server — it renders **Jinja2 HTML templates** that also include Vue.js loaded via a **CDN `<script>` tag**. Vue is used to progressively enhance specific parts of the page without replacing the server-rendering model.

**Architecture:**

```
Browser
  │
  ▼ HTTP Request
Flask Server
  │  Renders HTML template using Jinja2
  ▼
Jinja2 Template (HTML + Vue CDN script)
  │
  ▼ Browser receives full HTML
Vue.js initialises on the client and enhances specific elements
```

**Minimal Example:**

```python
# app.py
from flask import Flask, render_template, jsonify

app = Flask(__name__)

@app.route('/')
def index():
    # Flask renders the initial page
    return render_template('index.html')

@app.route('/api/students')
def get_students():
    students = [
        {"id": 1, "name": "Jane Doe", "gpa": 3.8},
        {"id": 2, "name": "John Smith", "gpa": 3.5},
    ]
    return jsonify(students)
```

```html
<!-- templates/index.html — Jinja2 template with Vue embedded -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Student Portal</title>
</head>
<body>

  <!-- The element Vue will control -->
  <div id="app">
    <h1>Students</h1>
    <ul>
      <!-- Vue template syntax — note: conflicts with Jinja2! (see below) -->
      <li v-for="student in students" :key="student.id">
        ${ student.name } — GPA: ${ student.gpa }
      </li>
    </ul>
  </div>

  <!-- Vue 3 loaded from CDN — no build step needed -->
  <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
  <script>
    const { createApp, ref, onMounted } = Vue;

    createApp({
      // Change delimiters to avoid collision with Jinja2's {{ }}
      compilerOptions: {
        delimiters: ['${', '}']
      },
      setup() {
        const students = ref([]);

        onMounted(async () => {
          const response = await fetch('/api/students');
          students.value = await response.json();
        });

        return { students };
      }
    }).mount('#app');
  </script>

</body>
</html>
```

> **The Delimiter Collision Problem**: Both Jinja2 and Vue use `{{ }}` as their template syntax. When Flask renders the Jinja2 template, it processes `{{ }}` first — Vue never sees its own template syntax.
>
> **Fix 1 — Change Vue's delimiters** (shown above): `compilerOptions: { delimiters: ['${', '}'] }`
>
> **Fix 2 — Jinja2 raw blocks**: Wrap Vue markup in `{% raw %}...{% endraw %}` so Jinja2 ignores it:
> ```html
> {% raw %}
>   <li v-for="s in students">{{ s.name }}</li>
> {% endraw %}
> ```

**When to use the Embedded Approach:**

| Situation | Reason |
|---|---|
| Existing Flask app you want to add interactivity to | Minimal refactoring — just add Vue to specific pages |
| Simple forms with real-time validation | Vue on one element, Flask handles everything else |
| Server-side rendering is important (SEO, auth) | Flask still controls page rendering and routing |
| Small team / rapid prototyping | No build toolchain to configure |
| Admin dashboards with complex widgets | Mix Flask page structure with Vue-powered data tables, charts |

**Limitations:**
- No hot-module replacement — must refresh the page to see changes
- No access to Vue's component ecosystem (Single File Components)
- No TypeScript support without significant tooling setup
- Jinja2 delimiter collisions require workarounds

---

### Pattern B: Decoupled SPA (Single Page Application) Approach

In this pattern, Vue is a **completely independent project** — built with **Vite** (modern) or **Vue CLI** (older). Flask becomes a **pure JSON REST API backend**. The two projects are entirely separate codebases that communicate only via HTTP requests.

**Architecture:**

```
                  Development:
┌──────────────────────────────────────────────────────────┐
│  Vue Dev Server (Vite)         Flask API Server           │
│  localhost:5173                localhost:5000              │
│                                                           │
│  Vite proxies /api/* ─────────────────────────────────▶   │
│  requests to Flask                                        │
└──────────────────────────────────────────────────────────┘

                  Production:
┌──────────────────────────────────────────────────────────┐
│  Vue build output (dist/)      Flask API Server           │
│  Served by Flask's static      Handles /api/* routes      │
│  file serving OR a CDN                                    │
│                                                           │
│  Browser ──▶ Flask/CDN (HTML)                             │
│  Browser ──▶ Flask API (/api/students)                    │
└──────────────────────────────────────────────────────────┘
```

**Setting up the Vue project (Vite):**

```bash
# Create a new Vue 3 project with Vite
npm create vite@latest student-frontend -- --template vue
cd student-frontend
npm install
npm run dev   # Starts dev server on localhost:5173
```

**Configuring Vite to Proxy API Requests to Flask:**

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      // Any request to /api/* during development gets forwarded to Flask
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      }
    }
  }
})
```

This means during development:
- `fetch('/api/students')` in Vue → Vite intercepts → forwards to `http://localhost:5000/api/students`
- No CORS issues during development (proxy handles it)
- Vue components can use clean `/api/...` paths without hardcoding `localhost:5000`

**A complete Vue component fetching from Flask:**

```vue
<!-- src/components/StudentList.vue -->
<template>
  <div class="student-list">
    <h1>Students</h1>

    <div v-if="loading" class="spinner">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <ul v-else>
      <li v-for="student in students" :key="student.id" class="student-card">
        <strong>{{ student.name }}</strong>
        <span class="gpa">GPA: {{ student.gpa.toFixed(2) }}</span>
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const students = ref([])
const loading = ref(true)
const error = ref(null)

onMounted(async () => {
  try {
    const response = await fetch('/api/students')
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    students.value = await response.json()
  } catch (e) {
    error.value = `Failed to load students: ${e.message}`
  } finally {
    loading.value = false
  }
})
</script>
```

**When to use the Decoupled SPA Approach:**

| Situation | Reason |
|---|---|
| Building a rich, highly interactive app | Full access to Vue's component ecosystem |
| Multiple frontends (web + mobile) consuming the same API | Flask API is completely reusable |
| Large team with frontend/backend specialists | Teams work independently, merge via API contract |
| TypeScript + modern tooling required | Full Vite/Webpack ecosystem available |
| Complex client-side routing (Vue Router) | SPA routing with dynamic pages |

---

### Pattern A vs. Pattern B: Quick Comparison

| Feature | Embedded (CDN) | Decoupled SPA (Vite) |
|---|---|---|
| **Setup complexity** | Low (just add `<script>` tag) | Medium (separate project, build tooling) |
| **Vue SFCs (`.vue` files)** | No | Yes |
| **TypeScript** | Limited | Full support |
| **Hot Module Replacement** | No | Yes (instant updates in dev) |
| **Build optimization** | No | Yes (tree shaking, code splitting) |
| **Jinja2 template collision** | Yes (needs workaround) | No (Vue is completely separate) |
| **SEO** | Good (Flask renders HTML) | Needs SSR or prerendering |
| **CORS** | Not needed (same origin) | Needed in production |
| **Best for** | Enhancing existing Flask pages | Greenfield app with rich UI |

---

## 5.2 Client-Server API Communication

### Flask as a Pure JSON API Backend

In the decoupled pattern, Flask's only job is to receive requests, run logic, and return JSON. It never generates HTML.

**Core Flask patterns for a JSON API:**

```python
from flask import Flask, jsonify, request, abort
from functools import wraps

app = Flask(__name__)

# ─── GET: Return a collection ──────────────────────────────────────────────
@app.route('/api/students', methods=['GET'])
def get_students():
    students = Student.query.all()
    return jsonify([s.to_dict() for s in students])   # 200 OK by default

# ─── GET: Return a single resource ─────────────────────────────────────────
@app.route('/api/students/<int:id>', methods=['GET'])
def get_student(id):
    student = Student.query.get(id)
    if student is None:
        abort(404)                     # Flask sends {"error": "Not Found"} with 404
    return jsonify(student.to_dict())

# ─── POST: Create a resource ───────────────────────────────────────────────
@app.route('/api/students', methods=['POST'])
def create_student():
    data = request.get_json()          # Parse JSON body
    if not data or 'name' not in data:
        abort(400, description="'name' is required")

    student = Student(name=data['name'], email=data.get('email'))
    db.session.add(student)
    db.session.commit()
    return jsonify(student.to_dict()), 201   # 201 Created

# ─── PATCH: Partial update ─────────────────────────────────────────────────
@app.route('/api/students/<int:id>', methods=['PATCH'])
def update_student(id):
    student = Student.query.get_or_404(id)
    data = request.get_json()
    if 'email' in data:
        student.email = data['email']
    if 'gpa' in data:
        student.gpa = data['gpa']
    db.session.commit()
    return jsonify(student.to_dict())

# ─── DELETE: Remove a resource ─────────────────────────────────────────────
@app.route('/api/students/<int:id>', methods=['DELETE'])
def delete_student(id):
    student = Student.query.get_or_404(id)
    db.session.delete(student)
    db.session.commit()
    return jsonify({"success": True, "deleted_id": id})

# ─── Consistent error responses ────────────────────────────────────────────
@app.errorhandler(404)
def not_found(e):
    return jsonify({"error": "Not Found", "message": str(e)}), 404

@app.errorhandler(400)
def bad_request(e):
    return jsonify({"error": "Bad Request", "message": str(e)}), 400
```

**Standard HTTP Status Codes to use:**

| Code | Meaning | When to Use |
|---|---|---|
| `200 OK` | Success | GET, PATCH, DELETE responses |
| `201 Created` | Resource created | POST responses on success |
| `400 Bad Request` | Invalid input | Missing fields, wrong data types |
| `401 Unauthorized` | Not authenticated | No token / expired token |
| `403 Forbidden` | Authenticated but not allowed | Correct token, wrong permissions |
| `404 Not Found` | Resource doesn't exist | `GET /students/999` when 999 doesn't exist |
| `422 Unprocessable Entity` | Validation failed | Correct format but invalid values |
| `500 Internal Server Error` | Server bug | Unhandled exception |

---

### Data Fetching in Vue: `fetch()` vs `axios`

#### Native `fetch()` API

Built into every modern browser — no installation needed.

```javascript
// Basic GET
const response = await fetch('/api/students')
const students = await response.json()

// POST with JSON body
const response = await fetch('/api/students', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Jane Doe', email: 'jane@uni.edu' })
})
const newStudent = await response.json()

// IMPORTANT: fetch() does NOT throw on HTTP error status codes!
// You must check response.ok manually:
if (!response.ok) {
  const err = await response.json()
  throw new Error(err.message || `HTTP ${response.status}`)
}
```

> **`fetch()` gotcha**: Unlike `axios`, native `fetch()` only throws on **network errors** (no internet, server unreachable). A `404` or `500` response is considered "successful" by `fetch()` — you must check `response.ok` yourself.

#### Axios

A third-party HTTP client library with a cleaner, more consistent API.

```bash
npm install axios
```

```javascript
import axios from 'axios'

// GET — auto-parses JSON, throws on error status codes
const { data: students } = await axios.get('/api/students')

// POST
const { data: newStudent } = await axios.post('/api/students', {
  name: 'Jane Doe',
  email: 'jane@uni.edu'
})

// PATCH
const { data: updated } = await axios.patch(`/api/students/${id}`, {
  email: 'new@uni.edu'
})

// DELETE
await axios.delete(`/api/students/${id}`)
```

**Key Axios advantages over `fetch()`:**

| Feature | `fetch()` | `axios` |
|---|---|---|
| Auto JSON parsing | ✅ (need `.json()`) | ✅ (automatic) |
| Throws on error status | ❌ (must check `response.ok`) | ✅ (auto throws on 4xx/5xx) |
| Request interceptors | ❌ | ✅ (add auth headers globally) |
| Response interceptors | ❌ | ✅ (handle errors globally) |
| Request cancellation | ✅ (AbortController) | ✅ (CancelToken / AbortController) |
| Upload progress | ❌ | ✅ |
| Bundle size | 0 KB (native) | ~14 KB gzipped |

**Creating a reusable Axios instance with base URL:**

```javascript
// src/api/client.js
import axios from 'axios'

const apiClient = axios.create({
  baseURL: '/api',                          // All requests prepend /api
  timeout: 10000,                           // 10 second timeout
  headers: { 'Content-Type': 'application/json' }
})

export default apiClient

// Usage in a component:
import apiClient from '@/api/client'
const { data } = await apiClient.get('/students')     // → GET /api/students
const { data } = await apiClient.get('/students/42')  // → GET /api/students/42
```

---

### Lifecycle Hooks for Data Loading

In Vue 3 (Composition API), data is typically fetched when the component **mounts** (appears in the DOM).

```vue
<script setup>
import { ref, onMounted, watch } from 'vue'
import apiClient from '@/api/client'

const props = defineProps(['studentId'])

const student = ref(null)
const loading = ref(false)
const error = ref(null)

// Fetch on initial mount
async function loadStudent(id) {
  loading.value = true
  error.value = null
  try {
    const { data } = await apiClient.get(`/students/${id}`)
    student.value = data
  } catch (e) {
    error.value = e.response?.data?.message || 'Failed to load student'
  } finally {
    loading.value = false
  }
}

onMounted(() => loadStudent(props.studentId))

// Re-fetch if the studentId prop changes (e.g., navigating between students)
watch(() => props.studentId, (newId) => loadStudent(newId))
</script>
```

**Lifecycle hook timing:**

```
Component created
      │
      ▼
beforeMount()   ← DOM not yet available; avoid DOM manipulation here
      │
      ▼
onMounted()     ← DOM is ready; this is where you fetch data
      │
      ▼  (on prop/data change)
onUpdated()     ← Re-renders complete
      │
      ▼  (component removed)
onUnmounted()   ← Clean up: cancel pending requests, remove event listeners
```

**Cancelling in-flight requests on unmount (best practice):**

```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const students = ref([])
let abortController = null

onMounted(async () => {
  abortController = new AbortController()
  try {
    const response = await fetch('/api/students', {
      signal: abortController.signal   // Attach cancellation signal
    })
    students.value = await response.json()
  } catch (e) {
    if (e.name !== 'AbortError') console.error(e)  // Ignore cancellation errors
  }
})

onUnmounted(() => {
  // Cancel the request if component unmounts before it completes
  // Prevents "setting state on unmounted component" warnings
  abortController?.abort()
})
</script>
```

---

## 5.3 Authentication & Session Handling

Authentication in a decoupled Vue + Flask app requires careful design because the browser and server are on different origins in development, and even in production when using separate services.

### Token-Based Auth (JWT) vs. Session Cookies

**Option 1: JWT Tokens stored in `localStorage` / `sessionStorage`**

```
Login Flow:
  Vue sends POST /api/auth/login { email, password }
  Flask verifies → returns { token: "eyJhbGci..." }
  Vue stores token in localStorage

Subsequent Requests:
  Vue reads token from localStorage
  Vue adds Authorization: Bearer <token> header to every request
  Flask verifies token signature → allows/denies
```

```javascript
// src/api/auth.js
export async function login(email, password) {
  const { data } = await apiClient.post('/auth/login', { email, password })
  localStorage.setItem('token', data.token)
  return data
}

export function logout() {
  localStorage.removeItem('token')
}

export function getToken() {
  return localStorage.getItem('token')
}
```

**Option 2: Session Cookies (HttpOnly)**

```
Login Flow:
  Vue sends POST /api/auth/login { email, password }
  Flask verifies → sets HttpOnly session cookie in response headers
  Browser stores cookie automatically

Subsequent Requests:
  Browser automatically sends cookie with every request to the same domain
  Flask reads session cookie → validates session
```

```python
# Flask session cookie approach
from flask import session

@app.route('/api/auth/login', methods=['POST'])
def login():
    data = request.get_json()
    user = User.query.filter_by(email=data['email']).first()
    if user and user.check_password(data['password']):
        session['user_id'] = user.id      # Flask session (server-side or signed cookie)
        return jsonify({"success": True, "user": user.to_dict()})
    return jsonify({"error": "Invalid credentials"}), 401
```

**JWT vs. Cookies Comparison:**

| Aspect | JWT in localStorage | HttpOnly Session Cookie |
|---|---|---|
| **XSS Vulnerability** | ⚠️ JS can read localStorage → token theft if XSS | ✅ HttpOnly = JS cannot read cookie |
| **CSRF Vulnerability** | ✅ Not sent automatically → no CSRF risk | ⚠️ Auto-sent → needs CSRF token protection |
| **Cross-origin (CORS)** | ✅ Works easily with Authorization header | ⚠️ Requires `credentials: 'include'` + CORS config |
| **Stateless** | ✅ Flask doesn't need to store anything | ❌ Flask needs session storage (DB or Redis) |
| **Token expiry / revocation** | ⚠️ Hard to revoke early (need a blocklist) | ✅ Easy (delete server-side session) |
| **Mobile app support** | ✅ Easy | ⚠️ Cookie handling varies |

> **Recommendation**: For a Vue + Flask SPA, **JWT with short expiry + refresh tokens** is the most common and practical approach. Use HttpOnly cookies to store the refresh token for extra security.

---

### Adding Auth Headers via Axios Interceptors

The cleanest pattern: configure Axios to **automatically attach the Authorization header** to every request — no need to add it manually in every component.

```javascript
// src/api/client.js
import axios from 'axios'
import { getToken, refreshToken, logout } from './auth'
import router from '@/router'

const apiClient = axios.create({
  baseURL: '/api',
  timeout: 10000,
})

// ─── Request Interceptor: Attach token to every outgoing request ─────────
apiClient.interceptors.request.use(
  (config) => {
    const token = getToken()
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  (error) => Promise.reject(error)
)

// ─── Response Interceptor: Handle expired tokens globally ────────────────
apiClient.interceptors.response.use(
  (response) => response,   // Pass through successful responses
  async (error) => {
    const originalRequest = error.config

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true   // Prevent infinite retry loop
      try {
        await refreshToken()          // Try to get a new access token
        return apiClient(originalRequest)  // Retry the original request
      } catch (refreshError) {
        logout()
        router.push('/login')         // Redirect to login if refresh fails
        return Promise.reject(refreshError)
      }
    }
    return Promise.reject(error)
  }
)

export default apiClient
```

With this in place, every component just calls `apiClient.get(...)` — auth is handled transparently.

---

### Securing Flask Endpoints

**Flask-JWT-Extended** is the standard library for JWT auth in Flask:

```bash
pip install flask-jwt-extended
```

```python
from flask import Flask, jsonify, request
from flask_jwt_extended import (
    JWTManager, create_access_token, create_refresh_token,
    jwt_required, get_jwt_identity, get_jwt
)
from datetime import timedelta

app = Flask(__name__)
app.config['JWT_SECRET_KEY'] = 'your-secret-key-here'      # Use env var in production!
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(minutes=15)
app.config['JWT_REFRESH_TOKEN_EXPIRES'] = timedelta(days=30)
jwt = JWTManager(app)

# ─── Login: Issue tokens ────────────────────────────────────────────────
@app.route('/api/auth/login', methods=['POST'])
def login():
    data = request.get_json()
    user = User.query.filter_by(email=data.get('email')).first()

    if not user or not user.check_password(data.get('password')):
        return jsonify({"error": "Invalid email or password"}), 401

    access_token = create_access_token(identity=str(user.id))
    refresh_token = create_refresh_token(identity=str(user.id))
    return jsonify(access_token=access_token, refresh_token=refresh_token)

# ─── Refresh: Issue new access token using refresh token ────────────────
@app.route('/api/auth/refresh', methods=['POST'])
@jwt_required(refresh=True)
def refresh():
    identity = get_jwt_identity()
    access_token = create_access_token(identity=identity)
    return jsonify(access_token=access_token)

# ─── Protected route: Require valid JWT ────────────────────────────────
@app.route('/api/students', methods=['GET'])
@jwt_required()
def get_students():
    current_user_id = get_jwt_identity()  # ID from the token's 'sub' claim
    students = Student.query.all()
    return jsonify([s.to_dict() for s in students])

# ─── Role-based access: Require specific claims ────────────────────────
@app.route('/api/admin/students', methods=['DELETE'])
@jwt_required()
def admin_delete_student():
    claims = get_jwt()                     # Get full JWT payload
    if claims.get('role') != 'admin':
        return jsonify({"error": "Admin access required"}), 403
    # ... delete logic
```

---

### Vue Route Guards

Protect client-side routes using **Vue Router navigation guards** — redirect unauthenticated users to the login page before they can access protected views.

```javascript
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import { getToken } from '@/api/auth'

const routes = [
  { path: '/login', component: () => import('@/views/LoginView.vue') },
  {
    path: '/dashboard',
    component: () => import('@/views/DashboardView.vue'),
    meta: { requiresAuth: true }     // Mark this route as protected
  },
  {
    path: '/students',
    component: () => import('@/views/StudentsView.vue'),
    meta: { requiresAuth: true }
  },
  {
    path: '/admin',
    component: () => import('@/views/AdminView.vue'),
    meta: { requiresAuth: true, requiresRole: 'admin' }
  },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

// ─── Global Navigation Guard ────────────────────────────────────────────
router.beforeEach((to, from, next) => {
  const token = getToken()
  const isAuthenticated = !!token

  if (to.meta.requiresAuth && !isAuthenticated) {
    // Redirect to login, remembering where they wanted to go
    next({ path: '/login', query: { redirect: to.fullPath } })
    return
  }

  if (to.meta.requiresRole) {
    // Decode JWT to check role claim (without verifying signature — just for UI routing)
    const payload = JSON.parse(atob(token.split('.')[1]))
    if (payload.role !== to.meta.requiresRole) {
      next({ path: '/dashboard' })   // Redirect — server will enforce it too
      return
    }
  }

  next()  // Allow navigation
})

export default router
```

> **Important**: Client-side route guards are a **UX convenience** — they prevent the user from loading the wrong page in the browser. They are **not a security measure**. All actual authorization must be enforced on the **Flask API side** using `@jwt_required()` and role checks.

---

### Complete Auth Flow: Vue + Flask Together

```
1. User visits /dashboard (protected)
   │
   ▼ Vue Router beforeEach guard fires
   │  No token found → redirect to /login

2. User submits login form
   │
   ▼ Vue sends POST /api/auth/login { email, password }
   │
   ▼ Flask verifies, returns { access_token, refresh_token }
   │
   ▼ Vue stores tokens, redirects to /dashboard

3. Vue Dashboard component mounts, calls GET /api/students
   │
   ▼ Axios request interceptor adds Authorization: Bearer <token>
   │
   ▼ Flask @jwt_required() verifies token signature
   │
   ▼ Flask returns student data → Vue renders it

4. Access token expires (15 minutes later)
   │
   ▼ Next API call returns 401 Unauthorized
   │
   ▼ Axios response interceptor catches 401
   │  Sends POST /api/auth/refresh with refresh token
   │
   ▼ Flask issues new access token
   │
   ▼ Axios retries the original request with new token
   │  Seamless — user never notices

5. User clicks Logout
   │
   ▼ Vue removes tokens from storage
   │
   ▼ Vue Router redirects to /login
   │  (Server-side, the old tokens simply expire — no revocation needed
   │   unless you maintain a blocklist)
```
