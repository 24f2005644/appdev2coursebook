# 3. Understanding State



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **3. Understanding State**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 3.1 What is State?

> **State** is the collection of internal details of a system, stored in memory at a given point in time.

In simpler terms: state is **all the data** that your application is currently "remembering" — everything that could influence what the user sees or what the system does next.

### The Memory Analogy
Think of state like a **snapshot** of the application at a specific moment. If you froze time and recorded every variable, every piece of data, every flag — that frozen picture is the state.

Change the state → the application behaves differently. Same state + same input → always the same behavior.

---

### Key Properties of State

#### ① Reproducibility
> **Same state + same input → identical output / behavior.**

This is the most important property of well-managed state. It means:
- The UI is **predictable** — given the same state, the same UI is always rendered.
- Bugs are **reproducible** — if you can recreate the state, you can recreate the bug.
- Testing becomes **reliable** — you can write tests by asserting output for a given state.

This directly ties back to `UI = f(state)` from Topic 2 — the rendering function `f` is (ideally) a **pure function**: no hidden side effects, no surprises.

```
State A  →  f(State A)  →  UI_A   (always the same)
State B  →  f(State B)  →  UI_B   (always the same)
```

#### ② Complexity
> State is an **inherent necessity** for any non-trivial web application.

A completely stateless app would be just a static HTML page — no interactivity, no personalization, no sessions. The moment you need:
- A user to log in
- A shopping cart
- A form that remembers what was typed

…you need state.

However, state also introduces **complexity**:
- More state = more possible combinations of UI conditions.
- State that is poorly managed leads to **inconsistent UIs**, stale data, and hard-to-trace bugs.
- This is why state management is one of the most discussed and debated topics in frontend engineering.

> Managing state well is not just a technical challenge — it's a *design* challenge.

---

## 3.2 Levels / Hierarchy of State

Not all state is created equal. State exists at different **scopes and lifetimes**, and understanding which level a piece of data belongs to helps you decide *where* and *how* to store and manage it.

There are three distinct levels, from broadest to most transient:

```
┌─────────────────────────────────────┐
│           System State              │  ← Broadest (entire database)
│  ┌──────────────────────────────┐   │
│  │      Application State       │   │  ← Per-user / per-session
│  │  ┌───────────────────────┐   │   │
│  │  │      UI State         │   │   │  ← Per-component / ephemeral
│  │  └───────────────────────┘   │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

### 3.2.1 System State

> **The complete, ground-truth data of the entire system — typically the backend database.**

| Property | Detail |
|---|---|
| **Scope** | The whole system / all users |
| **Where it lives** | Backend database / server |
| **Who owns it** | The server / backend |
| **Persistence** | Long-term, durable |
| **Independence** | Exists independently of any client or UI |

#### Characteristics
- **Comprehensive**: Contains *all* data for *all* users and entities in the system.
- **Large-scale**: Can be gigabytes or terabytes of data.
- **Persistent**: Survives browser refreshes, server restarts (with proper DB setup), and user sessions.
- **Client-independent**: The system state exists whether or not any user is currently using the app.

#### Examples
- The **complete product catalogue** of an e-commerce site (all SKUs, prices, inventory counts for all users).
- All **articles and posts** on a news platform or blog.
- All **courses, students, and grades** in a university's academic system (e.g., the IITM BS degree portal).
- All **user accounts** in the system.

> The frontend never holds or owns system state. It only **fetches subsets** of it as needed.

---

### 3.2.2 Application State

> **The subset of system data relevant to a specific user's current session or interaction.**

| Property | Detail |
|---|---|
| **Scope** | A single user's session or client instance |
| **Where it lives** | Client (browser memory, local storage) or server session |
| **Who owns it** | The client app (with server as source of truth) |
| **Persistence** | Session-length or longer (via tokens, cookies) |
| **Independence** | Tied to a specific user's activity |

#### Characteristics
- **Session-scoped**: Typically starts when a user logs in / opens the app and ends when they leave.
- **Interactive**: Driven by user actions — adding to cart, setting preferences, navigating.
- **Personalised**: Different users have different application state at the same time.

#### Examples
- A user's **shopping cart** — only their items, not everyone else's.
- **User preferences** — dark mode on/off, language setting, notification preferences.
- The user's **personalized feed** — filtered/ranked content based on their history.
- Which **dashboard widgets** are active or pinned for this user.
- Whether the user is **logged in** and their identity/role (authentication state).

> Think of application state as: *"What does this specific user's version of the app currently look like?"*

---

### 3.2.3 UI State (Ephemeral State)

> **Transient, short-lived state that describes the current visual/interactive condition of rendered components.**

| Property | Detail |
|---|---|
| **Scope** | A single component or view |
| **Where it lives** | Component memory (e.g., React `useState`, Flutter `StatefulWidget`) |
| **Who owns it** | The component itself |
| **Persistence** | Extremely short — lost on navigation or re-render |
| **Independence** | Purely visual; does not need to survive page reload |

#### Characteristics
- **Ephemeral**: The word literally means *"lasting for a very short time"*. This state is throwaway.
- **Local**: Belongs to a specific widget/component and has no meaning outside it.
- **Visual**: Describes *how something looks right now*, not what data the user has.
- **No persistence needed**: If the user navigates away and comes back, resetting this state is perfectly fine.

#### Examples
- A **loading spinner or skeleton screen** shown while data is being fetched.
- Which **tab is currently active** in a tab bar.
- Whether a **dropdown or modal is open**.
- The **current value in a search input** before the user submits.
- Whether a form field has **focus** (cursor blinking in it).
- A **tooltip** being shown on hover.

> UI state is the "working memory" of a component — it's needed right now, and can be safely forgotten the moment it's no longer visible.

---

## Putting It All Together — A Practical Example

Consider a **university course registration app** (like the IITM portal):

| State Level | Example in this App |
|---|---|
| **System State** | All courses offered, all registered students, all grades for all students |
| **Application State** | The courses *this specific student* is registered for, their academic history, their profile |
| **UI State** | Whether the "Register" confirmation modal is open, which semester tab is selected, whether the course list is loading |

---

## Summary

| Level | Scope | Lifetime | Stored In | Example |
|---|---|---|---|---|
| **System State** | Entire system | Permanent | Backend DB | All products in catalogue |
| **Application State** | Single user session | Session-long | Client / server session | User's cart, preferences |
| **UI State** | Single component | Seconds/milliseconds | Component memory | Modal open, loading spinner |

Key takeaway:
- **System state** = the world's data.
- **Application state** = the user's data for this session.
- **UI state** = the component's data for this moment.

Managing state effectively means recognizing which level a piece of data belongs to — and choosing the right tool and location to store it.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
