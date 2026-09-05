# 2. Form Validation

---

## 2.1 Fundamentals of Validation

### What is Form Validation?
Form validation is the process of **checking that user-submitted data meets required constraints** before it is processed or stored. It acts as a quality gate — ensuring data integrity at the point of entry.

### Why Validate?
- **Data integrity**: Prevent garbage data from entering your system (e.g., an empty email field, a negative age).
- **User experience**: Give immediate, helpful feedback so users can correct mistakes without waiting for a server response.
- **Security**: Defend against malformed or malicious inputs (though client-side validation alone is **never sufficient** for security).

### Constraint Criteria (Types of Validation Rules)

| Rule Type | Example |
|---|---|
| **Required** | Field must not be empty |
| **Type** | Must be a number, email format, date, etc. |
| **Length / Range** | Min/max characters, min/max numeric value |
| **Pattern / Regex** | Must match a specific format (e.g., phone number) |
| **Custom / Cross-field** | Password must match confirm-password field |
| **Domain / Business logic** | Email must be from a specific domain (e.g., `@iitm.ac.in`) |

---

### Client-side vs. Server-side Validation

This is one of the most important trade-offs in web development:

| Aspect | Client-side | Server-side |
|---|---|---|
| **Where it runs** | In the browser (JavaScript) | On the server (Python, Node.js, etc.) |
| **Speed / UX** | ✅ Instant feedback — no network wait | ❌ Requires a round-trip request |
| **Security** | ⚠️ **NOT secure** — can be bypassed by the user | ✅ **Authoritative** — cannot be bypassed |
| **Can be skipped?** | ✅ Yes — users can disable JS or send raw HTTP | ❌ No — always enforced |
| **Purpose** | Improving user experience | Enforcing data integrity and security |

> ⚠️ **Critical Rule**: **Never rely solely on client-side validation for security.** A malicious user can bypass JavaScript entirely using browser dev tools or tools like `curl`. Always re-validate on the server.

**Best practice**: Use **both**. Client-side for fast UX; server-side as the authoritative safety net.

---

## 2.2 Vue Reactivity & Form Handling

Vue's reactivity system makes form handling elegant — the UI and data are always in sync.

### Two-way Data Binding with `v-model`

`v-model` creates a **two-way binding** between a form input and a Vue data property. When the user types, the data updates. When the data changes, the input reflects it.

```vue
<template>
  <input v-model="email" type="email" placeholder="Enter email" />
  <p>You typed: {{ email }}</p>
</template>

<script>
export default {
  data() {
    return {
      email: ''
    };
  }
}
</script>
```

`v-model` is essentially syntactic sugar for:
```html
<input :value="email" @input="email = $event.target.value" />
```

#### `v-model` Modifiers

| Modifier | Behavior | Example |
|---|---|---|
| `.lazy` | Sync on `change` event (on blur), not on every keystroke | `v-model.lazy="name"` |
| `.number` | Auto-cast input to a Number | `v-model.number="age"` |
| `.trim` | Auto-strip leading/trailing whitespace | `v-model.trim="username"` |

---

### Conditional Error Rendering with `v-if` and `v-show`

Error messages should only appear when a field is invalid. Vue's directives make this straightforward:

```vue
<template>
  <div>
    <input v-model="email" type="text" placeholder="Email" />
    <!-- v-if: removes the element from the DOM entirely when false -->
    <p v-if="!isValidEmail" class="error">Please enter a valid email.</p>
  </div>
</template>

<script>
export default {
  data() {
    return { email: '' };
  },
  computed: {
    isValidEmail() {
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(this.email);
    }
  }
}
</script>
```

**`v-if` vs `v-show` for error messages:**

| | `v-if` | `v-show` |
|---|---|---|
| **DOM behavior** | Adds/removes the element entirely | Always in DOM; toggles `display: none` |
| **Performance** | Higher toggle cost (DOM manipulation) | Higher initial cost (renders element early) |
| **Best for** | Elements rarely shown | Elements toggled frequently |
| **Recommendation for errors** | ✅ Preferred — errors are infrequent |  |

---

### Intercepting Form Submission — `@submit.prevent`

By default, an HTML `<form>` submission causes a **full page reload** (native browser behavior). In a Vue SPA, you want to **prevent this** and handle submission in JavaScript.

```vue
<template>
  <!-- @submit.prevent = event.preventDefault() + call handleSubmit() -->
  <form @submit.prevent="handleSubmit">
    <input v-model="email" type="text" />
    <button type="submit">Submit</button>
  </form>
</template>

<script>
export default {
  data() {
    return { email: '' };
  },
  methods: {
    handleSubmit() {
      if (this.isValid()) {
        // Safe to process / send data to API
        console.log('Submitting:', this.email);
      }
    },
    isValid() {
      return this.email.includes('@');
    }
  }
}
</script>
```

**How `.prevent` works**: `@submit.prevent` is Vue's shorthand for calling `event.preventDefault()` on the submit event. This is equivalent to:
```js
// Without Vue shorthand:
<form @submit="(event) => { event.preventDefault(); handleSubmit(); }">
```

Other useful **event modifiers**:

| Modifier | Effect |
|---|---|
| `.prevent` | Calls `event.preventDefault()` |
| `.stop` | Calls `event.stopPropagation()` |
| `.once` | Handler fires only once |
| `.self` | Only fires if event target is the element itself |

---

## 2.3 Custom & Advanced Validation

### Custom Validation Rules

Beyond simple required/type checks, real-world apps need business-logic validations:

#### Regex Validation (Pattern Matching)
```vue
<script>
export default {
  data() {
    return { phone: '' };
  },
  computed: {
    phoneError() {
      const regex = /^[6-9]\d{9}$/; // Indian phone number format
      if (!this.phone) return 'Phone is required.';
      if (!regex.test(this.phone)) return 'Enter a valid 10-digit phone number.';
      return null; // no error
    }
  }
}
</script>
```

#### Domain / Business Logic Validation
```vue
computed: {
  emailError() {
    if (!this.email) return 'Email is required.';
    if (!this.email.endsWith('@iitm.ac.in')) return 'Must use an IITM email address.';
    return null;
  }
}
```

#### Cross-field / Aggregate Validation
```vue
data() {
  return { password: '', confirmPassword: '' };
},
computed: {
  passwordMatchError() {
    if (this.confirmPassword && this.password !== this.confirmPassword) {
      return 'Passwords do not match.';
    }
    return null;
  },
  // Aggregated: is the entire form valid?
  isFormValid() {
    return !this.emailError && !this.passwordMatchError && this.password.length >= 8;
  }
}
```

---

### Suppressing Browser's Native Validation — `novalidate`

Browsers have built-in form validation that shows native UI tooltips (e.g., "Please fill in this field"). These can **conflict with your custom Vue validation UI**.

To disable browser-native validation and fully control the experience:
```html
<form @submit.prevent="handleSubmit" novalidate>
  <!-- Now browser won't show its own tooltips; you control all error messages -->
  <input v-model="email" type="email" />
  <p v-if="emailError">{{ emailError }}</p>
</form>
```

> **Note**: Setting `novalidate` on the form **disables all HTML5 built-in validation**. You become fully responsible for validating every field.

---

### Complete Example — Putting It All Together

```vue
<template>
  <form @submit.prevent="handleSubmit" novalidate>
    <div>
      <label>Email</label>
      <input v-model.trim="email" type="text" placeholder="you@example.com" />
      <p v-if="emailError" class="error">{{ emailError }}</p>
    </div>

    <div>
      <label>Password</label>
      <input v-model="password" type="password" />
      <p v-if="passwordError" class="error">{{ passwordError }}</p>
    </div>

    <button type="submit" :disabled="!isFormValid">Register</button>
  </form>
</template>

<script>
export default {
  data() {
    return {
      email: '',
      password: '',
      submitted: false,
    };
  },
  computed: {
    emailError() {
      if (!this.email) return 'Email is required.';
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(this.email)) return 'Invalid email format.';
      return null;
    },
    passwordError() {
      if (!this.password) return 'Password is required.';
      if (this.password.length < 8) return 'Password must be at least 8 characters.';
      return null;
    },
    isFormValid() {
      return !this.emailError && !this.passwordError;
    }
  },
  methods: {
    handleSubmit() {
      this.submitted = true;
      if (this.isFormValid) {
        console.log('Form is valid — submitting!', { email: this.email });
      }
    }
  }
}
</script>

<style>
.error { color: red; font-size: 0.85rem; margin-top: 4px; }
</style>
```

---

## Summary

```
Form Validation in Vue
├── Why validate?         → Data integrity, UX, security
├── Client vs Server      → Client = UX speed; Server = security authority; use BOTH
├── v-model               → Two-way binding; modifiers: .lazy, .number, .trim
├── Error rendering       → v-if (preferred) / v-show to show/hide messages
├── @submit.prevent       → Stops page reload; gives control to Vue handler
├── novalidate            → Disables browser-native tooltips for full custom control
└── Custom rules          → Regex, domain checks, cross-field (computed properties)
```
