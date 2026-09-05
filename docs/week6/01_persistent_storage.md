# 1. Persistent Storage

---

## 1.1 Concept & Definition — *What is Persistent Storage?*

### The Problem
By default, Vue (like any JavaScript framework) stores application data **in memory** (RAM). This means:
- Every time a user **refreshes the page** or **closes the tab**, all Vue reactive data (`data()`, `ref()`, `reactive()`) is **destroyed and reset** to its initial state.
- There is **no memory** of what the user was doing — their inputs, preferences, or session-level changes are all lost.

### The Solution
**Persistent Storage** refers to mechanisms that allow data to **survive page reloads and browser sessions** by writing it to durable storage on the client's machine — without necessarily involving a backend server.

> **Key Insight**: Persistence ≠ a database. You can persist data entirely on the client side, inside the user's own browser.

---

## 1.2 Motivations & Use Cases — *Why use Client-Side Persistence?*

### Server-side Persistence vs. Client-side Persistence

| Aspect | Server-side | Client-side |
|---|---|---|
| **Storage location** | Remote database (PostgreSQL, MongoDB, etc.) | User's browser (localStorage, IndexedDB, cookies) |
| **Requires network?** | ✅ Yes — always needs a connection | ❌ No — works fully offline |
| **Latency** | Higher (network roundtrip) | None (instant read/write) |
| **Security** | More secure (data on your servers) | Less secure (user can inspect/modify) |
| **Data shared across devices?** | ✅ Yes | ❌ No (device-local only) |
| **Suitable for** | Multi-user apps, sensitive data | Preferences, caches, light local state |

### When to Use Client-Side Persistence

1. **Offline-first apps**: Applications that need to function without an active internet/server connection (e.g., a note-taking app, a to-do list).
2. **Performance optimization**: Avoid unnecessary network requests for data that doesn't change often (e.g., user theme preference, language setting).
3. **Simple apps without a backend**: Prototypes or tools where setting up a full server would be overkill.
4. **User preferences & local configuration**: Store UI settings (dark mode, font size, last visited page) that are personal to this device.

---

## 1.3 Mechanisms & Comparison — *How do we persist data?*

There are **three primary client-side storage mechanisms** available in the browser:

### 🍪 1. Cookies (`document.cookie`)

- **Original purpose**: Session management and server communication (sent with every HTTP request).
- **API**: Accessible via `document.cookie` in JavaScript. Typically wrapped in helper functions:
  ```js
  function setCookie(name, value, days) {
    document.cookie = `${name}=${value}; max-age=${days * 86400}`;
  }
  function getCookie(name) {
    return document.cookie.split('; ')
      .find(row => row.startsWith(name + '='))
      ?.split('=')[1];
  }
  ```
- **Lifespan**: Can be session-bound (expires on browser close) or set with an expiry date (`max-age` / `expires`).
- **Capacity**: Very small — typically **~4KB** per cookie.
- **Sent to server**: ⚠️ Automatically included in every HTTP request header — adds overhead and can be a security concern.
- **Best for**: Authentication tokens, session IDs (things the server needs to see).
- **Not ideal for**: Large data, pure client-side state.

---

### 📦 2. Web Storage API — `localStorage` & `sessionStorage`

Introduced in HTML5 as a cleaner, more capable alternative to cookies for client-side use.

#### `localStorage`
- **Scope**: Persists **indefinitely** until explicitly cleared by the user or the application.
- **Capacity**: ~**5MB** per origin (domain).
- **API**: Simple synchronous key-value store.
  ```js
  // Writing
  localStorage.setItem('username', 'vinay');

  // Reading
  const user = localStorage.getItem('username'); // "vinay"

  // Deleting
  localStorage.removeItem('username');

  // Clear all
  localStorage.clear();
  ```
- **Storing Objects**: localStorage only stores **strings**, so objects must be serialized:
  ```js
  // Saving an object
  const settings = { theme: 'dark', fontSize: 16 };
  localStorage.setItem('settings', JSON.stringify(settings));

  // Retrieving an object
  const savedSettings = JSON.parse(localStorage.getItem('settings'));
  ```

#### `sessionStorage`
- **Same API** as `localStorage`.
- **Key difference**: Data is **cleared when the browser tab is closed** (tab-scoped, session-only).
- Use when you want data to persist across page refreshes *within the same session* but not beyond.

| Feature | `localStorage` | `sessionStorage` |
|---|---|---|
| **Lifespan** | Permanent (until cleared) | Tab/session lifetime |
| **Scope** | All tabs on the same origin | Single tab only |
| **Capacity** | ~5MB | ~5MB |
| **Auto-sent to server?** | ❌ No | ❌ No |

---

### 🗄️ 3. IndexedDB

- A **low-level, transactional, object-oriented database** built into the browser.
- Unlike localStorage (key-string pairs), IndexedDB stores **structured JavaScript objects**.
- **Capacity**: Much larger — typically **hundreds of MB** or more (quota managed by browser).
- **Key concepts**:
  - **Object Store**: Analogous to a table in SQL — stores records identified by a **key path** (a property of the object, e.g., `id`).
  - **Transactions**: All reads and writes happen within transactions (ensures data integrity).
  - **Indexes**: You can create indexes on object properties for faster querying.
- **API** is low-level and callback/event-based (complex). Libraries like **Dexie.js** wrap it for easier use.
  ```js
  // Simplified example (raw IndexedDB)
  const request = indexedDB.open('MyAppDB', 1);
  request.onsuccess = (event) => {
    const db = event.target.result;
    const tx = db.transaction('notes', 'readwrite');
    const store = tx.objectStore('notes');
    store.add({ id: 1, title: 'Hello', body: 'World' });
  };
  ```
- **Best for**: Large datasets, complex querying, offline-capable apps with significant local data (e.g., a full PWA).

---

## 1.4 WebStorage API — Deep Dive & Vue Integration

### Storage Limits & Quota Handling

- Browsers enforce a **~5MB limit** per origin for Web Storage.
- Exceeding this limit throws a `QuotaExceededError` (a `DOMException`).
- Always wrap writes in a `try/catch` for production apps:
  ```js
  try {
    localStorage.setItem('largeData', bigString);
  } catch (e) {
    if (e.name === 'QuotaExceededError') {
      console.warn('Storage quota exceeded!');
    }
  }
  ```

### Integrating localStorage with Vue

A clean pattern is to sync Vue reactive state with `localStorage` using a **watcher**:

```vue
<script>
export default {
  data() {
    return {
      // Initialize from localStorage, fall back to default
      username: localStorage.getItem('username') || '',
    };
  },
  watch: {
    // Automatically save to localStorage whenever 'username' changes
    username(newValue) {
      localStorage.setItem('username', newValue);
    }
  }
}
</script>
```

For reactive objects, serialize with `JSON.stringify`:
```vue
<script>
export default {
  data() {
    return {
      settings: JSON.parse(localStorage.getItem('settings')) || { theme: 'light' },
    };
  },
  watch: {
    settings: {
      deep: true, // Watch nested properties
      handler(newValue) {
        localStorage.setItem('settings', JSON.stringify(newValue));
      }
    }
  }
}
</script>
```

> 📖 **See also**: [Vue.js Client-Side Storage Cookbook](https://vuejs.org/guide/best-practices/performance) — official patterns for working with browser storage in Vue apps.

---

## Summary

| Mechanism | Capacity | Lifespan | Complexity | Best Use |
|---|---|---|---|---|
| **Cookies** | ~4KB | Configurable | Low | Auth tokens, server communication |
| **localStorage** | ~5MB | Permanent | Very Low | Preferences, simple app state |
| **sessionStorage** | ~5MB | Tab session | Very Low | Temporary tab-scoped state |
| **IndexedDB** | Hundreds of MB | Permanent | High | Large data, offline apps |
