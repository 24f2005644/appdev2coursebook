# 4. Client-Side Data Fetching: Fetch API & Axios

---

## 4.1 The Fetch API

### Why Fetching a URL Must Be Asynchronous

Fetching data from a URL involves a full network round-trip:
1. DNS lookup to resolve the hostname to an IP address
2. TCP connection handshake (+ TLS handshake for HTTPS)
3. Sending the HTTP request
4. Waiting for the server to process and respond
5. Receiving and parsing the response body

None of these steps have predictable or guaranteed durations. Making them synchronous would freeze the browser's single main thread — blocking all rendering, clicks, and user interaction. This is why **`fetch()` is inherently asynchronous and returns a Promise** — it offloads the network I/O to browser background threads and notifies JavaScript only when the data is ready.

---

### What is the Fetch API?

The **Fetch API** is a modern, built-in browser standard for making HTTP requests from JavaScript. It was introduced in **ES6 (2015)** and is available globally in every modern browser as `window.fetch()` (or just `fetch()`).

It replaces the older, verbose `XMLHttpRequest (XHR)` API with a clean, Promise-based interface.

```
Old way:   XMLHttpRequest → verbose, callback-based, difficult to chain
New way:   fetch()        → clean, Promise-based, chainable with .then()
```

---

### The `fetch()` Interface

The simplest possible call:

```js
fetch('https://api.example.com/data')
```

This returns a **Promise** that resolves to a **`Response`** object — not the data itself, just the HTTP response envelope (status, headers, body stream).

#### Step 1: Get the Response

```js
fetch('https://hacker-news.firebaseio.com/v0/topstories.json')
    .then(function(response) {
        // response is a Response object — NOT the data yet
        console.log(response.status);      // 200
        console.log(response.ok);          // true (status 200-299)
        console.log(response.headers.get('Content-Type'));  // 'application/json'

        // To get the actual data, we must parse the body — also async!
        return response.json();            // Returns another Promise
    })
    .then(function(data) {
        // NOW we have the parsed JavaScript object/array
        console.log(data);
    })
    .catch(function(error) {
        console.error('Fetch failed:', error);
    });
```

> **Common gotcha**: `fetch()` only rejects on **network failure** (no connection, DNS failure). HTTP error responses like `404 Not Found` or `500 Server Error` still **resolve** — you must manually check `response.ok` to detect them.

```js
fetch('/api/user/999')
    .then(function(response) {
        if (!response.ok) {
            // Must manually throw — fetch won't reject on 404/500
            throw new Error(`HTTP Error: ${response.status} ${response.statusText}`);
        }
        return response.json();
    })
    .then(data => console.log(data))
    .catch(err => console.error(err.message));  // 'HTTP Error: 404 Not Found'
```

---

### Making Different HTTP Requests

`fetch()` accepts an optional second argument — an options object — for controlling method, headers, and body:

```js
// GET (default)
fetch('/api/articles');

// POST with JSON body
fetch('/api/articles', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer my-token'
    },
    body: JSON.stringify({ title: 'My Article', content: 'Hello World' })
});

// PUT — update a resource
fetch('/api/articles/42', {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title: 'Updated Title' })
});

// DELETE
fetch('/api/articles/42', { method: 'DELETE' });
```

---

### Using `fetch()` with `async`/`await`

The cleaner, modern way to use `fetch()`:

```js
async function getTopStories() {
    try {
        const response = await fetch('https://hacker-news.firebaseio.com/v0/topstories.json');

        if (!response.ok) {
            throw new Error(`Request failed: ${response.status}`);
        }

        const ids = await response.json();        // Parse JSON body — also awaited
        console.log('Top story IDs:', ids.slice(0, 5));
    } catch (error) {
        console.error('Error:', error.message);
    }
}

getTopStories();
```

---

### Response Body Parsing Methods

The `Response` object exposes different methods to parse the body depending on the expected format:

| Method | Returns | Use When |
|---|---|---|
| `response.json()` | Parsed JavaScript object | API returns JSON data |
| `response.text()` | Raw string | API returns plain text or HTML |
| `response.blob()` | Binary `Blob` object | API returns images, files, or binary data |
| `response.arrayBuffer()` | Raw `ArrayBuffer` | Low-level binary data |
| `response.formData()` | `FormData` object | Multipart form data |

All of these return a **Promise** — they must be awaited or chained with `.then()`.

---

### Browser Compatibility and Polyfills

`fetch()` is supported in all modern browsers (Chrome, Firefox, Safari, Edge). However, for legacy browser support (e.g., IE11):
- A **polyfill** such as `whatwg-fetch` or `cross-fetch` can be used
- These libraries implement the same `fetch()` API and transparently fall back to `XMLHttpRequest` in older browsers

```js
// Installing a polyfill (for legacy support)
// npm install whatwg-fetch
import 'whatwg-fetch';    // Adds fetch() to window in environments that lack it

// Now fetch() works even in IE11
fetch('/api/data').then(r => r.json()).then(console.log);
```

---

## 4.2 Axios Library

### What is Axios?

**Axios** is a popular, third-party HTTP client library for JavaScript. Unlike `fetch()`, which is a browser standard, Axios is an independent library that must be installed separately.

```bash
# Install via npm
npm install axios

# Or include via CDN in a plain HTML file
# <script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
```

---

### Cross-Environment Compatibility

One of Axios's most significant advantages is that it is **isomorphic** — it runs **identically** in both:
- **Browser environments**: Uses `XMLHttpRequest` under the hood
- **Node.js environments**: Uses Node's built-in `http`/`https` modules

This means the exact same Axios code you write for a Vue.js frontend works without modification in a Node.js backend script, a server-side rendering setup, or a test environment.

```js
// This exact code works in both a browser Vue component AND a Node.js script:
import axios from 'axios';

const response = await axios.get('https://api.example.com/data');
console.log(response.data);
```

Native `fetch()` is not available in older Node.js versions (added only in Node 18+), so Axios was historically the go-to choice for universal JS code.

---

### Automatic JSON Transformation

This is Axios's most developer-friendly feature:

| Step | `fetch()` | `axios` |
|---|---|---|
| **Sending JSON** | Must manually `JSON.stringify(body)` and set `Content-Type` header | Automatically serializes objects to JSON and sets the header |
| **Receiving JSON** | Must call `response.json()` (returns another Promise) | Automatically parses JSON; data is directly in `response.data` |

```js
// With fetch() — 2 steps to get JSON
const response = await fetch('/api/user/1');
const user = await response.json();     // Extra step!
console.log(user.name);

// With axios — data is immediately available
const response = await axios.get('/api/user/1');
console.log(response.data.name);        // No extra parsing step
```

```js
// Sending data with fetch() — manual work
await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },   // Must set manually
    body: JSON.stringify({ name: 'Alice' })             // Must serialize manually
});

// Sending data with axios — automatic
await axios.post('/api/users', { name: 'Alice' });      // Object passed directly
```

---

### Backward Compatibility

- Axios supports older browsers including **Internet Explorer 11** natively, without requiring a polyfill
- It works with Node.js versions going back further than native `fetch()` support
- This made it the default choice in Vue CLI-generated projects for years

---

### `fetch()` vs `axios` — Key Differences & Trade-offs

| Feature | `fetch()` | `axios` |
|---|---|---|
| **Built into browser?** | ✅ Yes — no installation needed | ❌ No — must `npm install axios` |
| **Node.js support** | ⚠️ Only Node 18+ natively | ✅ All versions (uses http/https module) |
| **JSON auto-parsing** | ❌ Must call `response.json()` | ✅ Auto-parsed into `response.data` |
| **JSON auto-serialization** | ❌ Must `JSON.stringify()` + set header | ✅ Automatic |
| **HTTP error handling** | ❌ Only rejects on network failure, not HTTP errors | ✅ Rejects on HTTP error status codes (4xx, 5xx) |
| **Request cancellation** | `AbortController` API | `AbortSignal` / `CancelToken` (deprecated) |
| **Request interceptors** | ❌ Not built-in | ✅ `axios.interceptors.request.use()` |
| **Response interceptors** | ❌ Not built-in | ✅ `axios.interceptors.response.use()` |
| **Download/Upload progress** | ❌ Limited | ✅ `onUploadProgress`, `onDownloadProgress` |
| **Bundle size** | Zero (native) | ~14KB (minified + gzipped) |
| **Legacy browser support** | Needs polyfill for IE11 | ✅ Built-in |

---

### Axios in Practice

```js
import axios from 'axios';

// GET request
async function fetchUser(id) {
    try {
        const response = await axios.get(`/api/users/${id}`);
        return response.data;          // Data is directly here — no .json() needed
    } catch (error) {
        if (error.response) {
            // Server responded with a 4xx or 5xx status
            console.error('Server error:', error.response.status, error.response.data);
        } else if (error.request) {
            // Request was made but no response received (network failure)
            console.error('Network error:', error.request);
        } else {
            console.error('Error:', error.message);
        }
    }
}

// POST request
async function createUser(userData) {
    const response = await axios.post('/api/users', userData);  // No stringify needed
    return response.data;
}

// Axios instance with a base URL — useful for Vue apps
const api = axios.create({
    baseURL: 'https://api.example.com/v1',
    timeout: 5000,
    headers: { 'Authorization': 'Bearer my-token' }
});

// All requests now use the base URL
const user = await api.get('/users/1');   // calls https://api.example.com/v1/users/1
```

---

### Axios Interceptors

Interceptors let you run code globally before every request is sent or after every response is received — perfect for attaching auth tokens or handling 401 errors globally:

```js
// Add auth token to every outgoing request
axios.interceptors.request.use(function(config) {
    const token = localStorage.getItem('auth_token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});

// Handle 401 Unauthorized globally (e.g., redirect to login)
axios.interceptors.response.use(
    function(response) {
        return response;   // Pass through successful responses
    },
    function(error) {
        if (error.response && error.response.status === 401) {
            window.location.href = '/login';   // Redirect to login page
        }
        return Promise.reject(error);
    }
);
```

---

## 4.3 Consuming Existing / Third-Party APIs

### The Concept: Rich Frontends Without a Custom Backend

One of the most powerful implications of the decoupled frontend architecture is that **you can build a fully functional, data-rich application entirely in the browser by consuming existing public APIs** — no custom backend required.

The public web exposes a vast ecosystem of free APIs that provide real-world, live data:

---

### Public API Ecosystem & Examples

#### 🌤️ OpenWeatherMap API
Provides current weather, forecasts, historical data for any city worldwide.

```js
const API_KEY = 'your_api_key';
const city = 'Mumbai';

async function getWeather() {
    const res = await axios.get('https://api.openweathermap.org/data/2.5/weather', {
        params: {
            q: city,
            appid: API_KEY,
            units: 'metric'    // Celsius
        }
    });

    const weather = res.data;
    console.log(`${weather.name}: ${weather.main.temp}°C, ${weather.weather[0].description}`);
    // Mumbai: 28.4°C, few clouds
}
```

---

#### 📰 HackerNews API
Free, no-auth-required API for tech news stories and discussions from news.ycombinator.com.

```js
async function getTopStories() {
    // Step 1: Get list of top story IDs
    const { data: ids } = await axios.get(
        'https://hacker-news.firebaseio.com/v0/topstories.json'
    );

    // Step 2: Fetch details for the first 5 stories in parallel
    const top5 = await Promise.all(
        ids.slice(0, 5).map(id =>
            axios.get(`https://hacker-news.firebaseio.com/v0/item/${id}.json`)
                 .then(res => res.data)
        )
    );

    top5.forEach(story => {
        console.log(`[${story.score}] ${story.title} — ${story.url}`);
    });
}
```

---

#### 📖 Wikipedia API
Access Wikipedia article summaries, search results, and content in any language.

```js
async function searchWikipedia(query) {
    const res = await axios.get('https://en.wikipedia.org/w/api.php', {
        params: {
            action: 'query',
            list: 'search',
            srsearch: query,
            format: 'json',
            origin: '*'    // Required for CORS
        }
    });

    const results = res.data.query.search;
    results.forEach(article => {
        console.log(`📄 ${article.title}: ${article.snippet.replace(/<[^>]+>/g, '')}`);
    });
}

searchWikipedia('Vue.js');
```

---

#### 🐙 GitHub API
Access public repository data, user profiles, commits, issues, pull requests.

```js
async function getUserRepos(username) {
    const res = await axios.get(`https://api.github.com/users/${username}/repos`, {
        params: { sort: 'stars', direction: 'desc', per_page: 5 }
    });

    res.data.forEach(repo => {
        console.log(`⭐ ${repo.stargazers_count} — ${repo.full_name}: ${repo.description}`);
    });
}

getUserRepos('vuejs');
// ⭐ 206000 — vuejs/vue: The Progressive JavaScript Framework
// ⭐ 43000  — vuejs/core: 🖖 Vue.js is a progressive, ...
```

---

### Integrating a Third-Party API in a Vue Component

This is a complete, practical Vue component that fetches GitHub user data on load:

```html
<template>
  <div class="github-card">
    <!-- Loading state -->
    <div v-if="isLoading">Loading GitHub profile...</div>

    <!-- Error state -->
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Data state -->
    <div v-else-if="user">
      <img :src="user.avatar_url" :alt="user.login" width="80" />
      <h2>{{ user.name }}</h2>
      <p>{{ user.bio }}</p>
      <p>📦 {{ user.public_repos }} public repos &nbsp; ⭐ {{ user.followers }} followers</p>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      user: null,
      isLoading: true,
      error: null
    };
  },

  async created() {
    try {
      const response = await axios.get('https://api.github.com/users/vuejs');
      this.user = response.data;
    } catch (err) {
      this.error = 'Failed to load GitHub profile: ' + err.message;
    } finally {
      this.isLoading = false;   // Always runs — hides the spinner
    }
  }
};
</script>
```

The three-state pattern (`isLoading`, `error`, `data`) ensures the UI always has something sensible to show, regardless of the outcome.

---

## Summary

```
Topic 4: Client-Side Data Fetching — Fetch API & Axios
│
├── 4.1 Fetch API
│     ├── Why async     ──► Network round-trips are unpredictable; blocking freezes the UI
│     ├── What it is    ──► Native browser standard (ES6); globally available as window.fetch()
│     ├── Interface     ──► Returns Promise<Response>; body must be parsed with .json(), .text(), etc.
│     ├── Error gotcha  ──► Only rejects on network failure; must manually check response.ok for HTTP errors
│     ├── HTTP methods  ──► GET (default), POST/PUT/DELETE via options object
│     └── Polyfills     ──► whatwg-fetch / cross-fetch for IE11 / older environments
│
├── 4.2 Axios Library
│     ├── What it is    ──► Third-party HTTP client; npm install axios
│     ├── Isomorphic    ──► Same code runs in browser AND Node.js (all versions)
│     ├── Auto JSON     ──► Serializes requests + parses responses automatically (response.data)
│     ├── Error model   ──► Rejects on 4xx/5xx HTTP errors (unlike fetch)
│     ├── Interceptors  ──► Global request/response hooks (auth tokens, 401 redirects)
│     └── vs fetch      ──► More features, more convenience; fetch is lighter, native
│
└── 4.3 Third-Party APIs
      ├── Concept      ──► Build rich apps using public APIs without writing a backend
      ├── OpenWeatherMap ─► Weather data by city, units, forecast
      ├── HackerNews   ──► Tech news, no auth required, Firebase-hosted
      ├── Wikipedia    ──► Article search, summaries in any language (CORS via origin=*)
      ├── GitHub       ──► Repos, users, stars, issues — rich public data
      └── Vue Pattern  ──► isLoading / error / data 3-state pattern in created() hook
```
