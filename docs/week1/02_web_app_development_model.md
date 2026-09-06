# Topic 2: Review of the Web Application Development Model



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 2: Review of the Web Application Development Model**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

> **MAD-II Week 1 | Detailed Notes**

---

## 2.1 Presentation & Logic Layers

Modern web applications are built with a **clear separation between what the user sees and what processes the data**. This separation is fundamental to maintainability and scalability.

### The Presentation Layer

The presentation layer is responsible for **how information is displayed** to the user. It lives entirely in the browser (client-side).

#### HTML — Semantic Markup
- **HTML (HyperText Markup Language)** provides the **structure and meaning** of content
- "Semantic" means tags convey meaning, not just appearance:

| Tag | Semantic Meaning |
|---|---|
| `<h1>` – `<h6>` | Headings with hierarchy |
| `<nav>` | Navigation section |
| `<article>` | Self-contained content block |
| `<section>` | Thematic grouping |
| `<header>` / `<footer>` | Page/section header and footer |
| `<form>` | User input collection |
| `<button>` | Clickable action trigger |

- **Non-semantic** (avoid overusing): `<div>`, `<span>` — have no meaning, just group elements
- Proper semantic HTML improves: **accessibility (screen readers)**, **SEO (search indexing)**, and **maintainability**

#### CSS — Layout and Styling
- **CSS (Cascading Style Sheets)** controls **visual presentation**: colors, fonts, spacing, layout, animations
- "Cascading" means styles are applied in a specific priority order (specificity rules)
- Key layout systems:
  - **Flexbox**: One-dimensional layout (rows OR columns)
  - **CSS Grid**: Two-dimensional layout (rows AND columns simultaneously)
- CSS is intentionally decoupled from HTML — the same HTML can look completely different with different stylesheets
- This enables **themes**, **responsive design** (different layouts for mobile vs. desktop), and **design consistency**

### The Logic Layer

The logic layer is responsible for **what the application does** — processing, storing, and serving data. It runs on a server (server-side).

#### Python with Flask
- **Flask** is a lightweight Python web microframework
- "Micro" means it provides the essentials without forcing a specific project structure or including heavy extras by default
- Core Flask responsibilities:
  - **Routing**: Mapping URLs to Python functions
    ```python
    @app.route('/users/<int:user_id>')
    def get_user(user_id):
        user = User.query.get(user_id)
        return render_template('user.html', user=user)
    ```
  - **Request handling**: Reading form data, JSON payloads, query parameters
  - **Response generation**: Rendering HTML templates (Jinja2) or returning JSON
  - **Database interaction**: Via ORM (SQLAlchemy) or raw SQL queries
  - **Session management**: Maintaining user state across requests

#### Why "Flexible" Backend?
- The course uses Flask as the example, but the principles apply universally
- The backend can be swapped — React frontend + Flask backend, or Vue frontend + Django backend, etc.
- This flexibility is a key strength of the layered architecture

---

## 2.2 Application Architecture

### The MVC Pattern (Model-View-Controller)

**MVC** is a design pattern that organizes a web application into **three distinct responsibilities**:

```
             USER INTERACTION
                    │
                    ▼
         ┌─────────────────────┐
         │    CONTROLLER       │  ← Receives input, orchestrates flow
         │  (Flask Routes /    │
         │   Request Handlers) │
         └────────┬────────────┘
                  │              │
         Updates  │              │  Queries / Updates
                  ▼              ▼
         ┌──────────────┐  ┌──────────────┐
         │     VIEW     │  │    MODEL     │
         │  (Templates, │  │  (Database,  │
         │   HTML/CSS)  │  │   Business   │
         │              │  │    Logic)    │
         └──────────────┘  └──────────────┘
                  │
                  ▼
              USER SEES
            (rendered page)
```

### Breaking Down Each Component

#### Model — Data & Business Logic
- Represents the **data structure** and **business rules** of the application
- Interacts directly with the database
- Contains validation, computations, relationships
- Example (Flask-SQLAlchemy):
  ```python
  class User(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      username = db.Column(db.String(80), unique=True, nullable=False)
      email = db.Column(db.String(120), unique=True, nullable=False)
      posts = db.relationship('Post', backref='author', lazy=True)
  ```

#### View — Presentation
- Responsible for **rendering data** into a format the user can see (HTML pages, JSON responses)
- Should contain **no business logic** — only display logic
- In Flask, views are Jinja2 templates:
  ```html
  <!-- user_profile.html -->
  <h1>{{ user.username }}</h1>
  <ul>
    {% for post in user.posts %}
      <li>{{ post.title }}</li>
    {% endfor %}
  </ul>
  ```

#### Controller — Request Orchestration
- **Receives user requests**, decides what to do, and **coordinates** between Model and View
- The "traffic controller" of the application
- In Flask, controllers are route handler functions:
  ```python
  @app.route('/user/<int:id>')
  def user_profile(id):
      # 1. Get data from Model
      user = User.query.get_or_404(id)
      # 2. Pass to View for rendering
      return render_template('user_profile.html', user=user)
  ```

### Why MVC? The Three Core Benefits

| Benefit | What it Means |
|---|---|
| **Separation of Concerns** | Each component has one job — easier to understand and debug |
| **Flexibility** | Swap out the View (e.g., change from HTML to JSON for APIs) without touching the Model |
| **Maintainability** | Changes in one layer rarely break another; teams can work in parallel |

> **Analogy**: Think of a restaurant — the **Kitchen (Model)** prepares the food (data), the **Waiter (Controller)** takes orders and delivers food, and the **Table Setting/Menu (View)** is what the customer interacts with. Each role is distinct.

---

## 2.3 System & API Architecture

### REST Principles & Sessions

#### HTTP is Stateless by Design
- Every HTTP request is **completely independent** — the server has no memory of previous requests
- Problem: Web apps need to maintain **stateful experiences** (e.g., logged-in users, shopping carts)
- Solution: **Sessions & Cookies**
  - The server creates a **session** upon login, stores session data server-side
  - A **session ID** is stored in a cookie on the client's browser
  - Every subsequent request sends the cookie, allowing the server to identify the user
  - Result: Stateful user experience over a stateless protocol

#### REST (Representational State Transfer)
REST is an **architectural style** for designing web APIs, not a protocol or standard. Key principles:

| Constraint | Description |
|---|---|
| **Stateless** | Each request must contain all information needed — no server-side session state for API calls |
| **Client-Server** | Clear separation; client and server evolve independently |
| **Uniform Interface** | Standard HTTP methods for predictable interactions |
| **Resource-Based** | Everything is a "resource" identified by a URL |
| **Cacheable** | Responses can be cached to improve performance |
| **Layered System** | Client doesn't know if it's talking directly to the server or through proxies |

#### HTTP Methods in REST

| Method | Action | Example |
|---|---|---|
| `GET` | Read / Retrieve | `GET /users/42` → fetch user with ID 42 |
| `POST` | Create | `POST /users` → create a new user |
| `PUT` | Replace (full update) | `PUT /users/42` → replace all data for user 42 |
| `PATCH` | Partial update | `PATCH /users/42` → update only specific fields |
| `DELETE` | Remove | `DELETE /users/42` → delete user 42 |

### APIs: Decoupling Data from Presentation

An **API (Application Programming Interface)** in web terms typically means an **HTTP endpoint that returns structured data** (usually JSON) rather than an HTML page.

#### Why Decouple?
- **Traditional approach**: Flask route → renders HTML → sends complete page to browser
- **API approach**: Flask route → returns JSON → client (browser/app) decides how to display it

**Traditional (Tightly coupled):**
```
Client Request → Server → Query DB → Render HTML → Send HTML page
```

**API-based (Decoupled):**
```
Client Request → Server → Query DB → Return JSON
                                         │
                          ┌──────────────┼──────────────┐
                          ▼              ▼               ▼
                     Browser JS      Mobile App     Another API
                     (renders UI)    (renders UI)   (processes data)
```

#### Benefits of Decoupling
- **Multiple clients, one backend**: Same API serves web app, iOS app, Android app
- **Independent evolution**: Update the frontend design without changing backend logic
- **Third-party integrations**: External services can consume your API
- **Testability**: APIs can be tested independently with tools like Postman or curl

#### JSON: The Universal Data Format
```json
{
  "id": 42,
  "username": "vinay_m",
  "email": "vinay@example.com",
  "posts": [
    {"id": 1, "title": "My First Post", "created_at": "2026-09-01"},
    {"id": 2, "title": "Learning MAD-II", "created_at": "2026-09-04"}
  ]
}
```

### Pragmatic RESTful APIs vs. Strict REST

**Strictly RESTful** APIs follow all REST constraints perfectly. In practice, most real-world APIs are **"RESTful" (pragmatic REST)** — they use HTTP methods and resource-based URLs but don't strictly adhere to every constraint.

| Strict REST | Pragmatic REST (Common in practice) |
|---|---|
| Truly stateless (no sessions) | May use session-based auth for web clients |
| HATEOAS (links in responses) | Usually omits HATEOAS — just returns data |
| Strict resource URLs | May include action-based endpoints like `/login`, `/search` |
| Strict status code usage | May use `200 OK` for everything with error messages in body |

> The course acknowledges this pragmatism — real applications often blend REST principles with practical needs. **Dogmatic adherence to REST is less important than building systems that work well.**

---

## 2.4 Supporting Concerns

These are the "non-functional" aspects of a web application that are essential for production systems.

### Authentication vs. Authorization

These are often confused but are distinct concepts:

| Concept | Question Answered | Example |
|---|---|---|
| **Authentication** | *"Who are you?"* | Verifying username + password to confirm identity |
| **Authorization** | *"What can you do?"* | Checking if a user has permission to delete a post |

- **Authentication** comes first — you must know who someone is before deciding what they can do
- Common authentication mechanisms:
  - Password-based login (with hashing — never store plain text passwords!)
  - Token-based auth (JWT — JSON Web Tokens)
  - OAuth2 (login with Google/GitHub)
  - Multi-factor authentication (MFA)

### Role-Based Access Control (RBAC)

**RBAC** is a model for managing authorization at scale:
- Users are assigned **roles** (e.g., Admin, Editor, Viewer)
- Roles are granted **permissions** (e.g., `can_delete_post`, `can_view_analytics`)
- Users inherit all permissions of their assigned role(s)

```
USER ──── assigned to ────► ROLE ──── has ────► PERMISSIONS
                             │
                    (Admin, Editor, Viewer)
                             │
                    Admin: [read, write, delete, manage_users]
                    Editor: [read, write]
                    Viewer: [read]
```

**Benefits of RBAC:**
- Scalable: Add new users to existing roles rather than configuring per-user permissions
- Auditable: Easy to review what each role can do
- Flexible: One user can have multiple roles

### Security

Security is not an afterthought — it must be built in from the start:

| Threat | Description | Mitigation |
|---|---|---|
| **SQL Injection** | Attacker inserts malicious SQL via input fields | Use ORMs / parameterized queries |
| **XSS (Cross-Site Scripting)** | Injecting malicious scripts into web pages | Escape output, use Content Security Policy |
| **CSRF (Cross-Site Request Forgery)** | Tricking users into making unintended requests | CSRF tokens on forms |
| **Broken Authentication** | Weak passwords, exposed sessions | Hashing (bcrypt), HTTPS, token expiry |
| **Sensitive Data Exposure** | Unencrypted data in transit or at rest | HTTPS, encrypted databases |

### Input Validation

**Never trust user input.** Validation must happen at multiple levels:

```
User Input
    │
    ├── Client-side validation (HTML5 / JS) ← UX only, easily bypassed
    │
    └── Server-side validation (Flask/Python) ← The actual security boundary
            │
            ├── Type checking (is it an integer?)
            ├── Range checking (is the value between 1-100?)
            ├── Format checking (is it a valid email?)
            └── Business rule checking (is the username unique?)
```

> **Rule**: Client-side validation is for user experience. Server-side validation is for security. You must always have both.

### Database Schema Design

A well-designed schema is foundational:
- **Normalization**: Eliminating data redundancy (1NF, 2NF, 3NF)
- **Indexes**: Speeding up common queries at the cost of write performance
- **Constraints**: Enforcing data integrity at the database level (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`)
- **Migrations**: Managing schema changes over time (Flask-Migrate / Alembic)

### Frontend Stack Selection

Choosing the right frontend technology involves trade-offs:

| Stack | Best For | Trade-offs |
|---|---|---|
| Vanilla HTML/CSS/JS | Simple sites, rapid prototyping | Limited scalability for complex UIs |
| Jinja2 Templates (Flask) | Server-rendered pages, SEO-critical content | Full page reloads, less dynamic |
| Vue.js / React / Angular | SPAs, rich interactive UIs | More complexity, JS bundle size |
| JAMStack | High-performance, CDN-delivered sites | Learning curve, build process |

---

## Summary of Topic 2

```
WEB APP DEVELOPMENT MODEL
│
├── PRESENTATION LAYER (Frontend)
│     ├── HTML: Semantic markup (structure & meaning)
│     └── CSS: Layout, styling, responsiveness
│
├── LOGIC LAYER (Backend - Flask/Python)
│     ├── Routing (URLs → Functions)
│     ├── Request/Response handling
│     └── Database interaction
│
├── ARCHITECTURE: MVC
│     ├── Model → Data + Business Logic
│     ├── View → Rendering (HTML Templates / JSON)
│     └── Controller → Orchestration (Route Handlers)
│
├── SYSTEM & API ARCHITECTURE
│     ├── HTTP Statelessness → Solved by Sessions & Cookies
│     ├── REST Principles (Stateless, Resource-based, Uniform Interface)
│     ├── APIs: Decouple data (JSON) from presentation
│     └── Pragmatic REST (practical > dogmatic)
│
└── SUPPORTING CONCERNS
      ├── Authentication ("Who are you?") vs Authorization ("What can you do?")
      ├── RBAC: Users → Roles → Permissions
      ├── Security: SQL Injection, XSS, CSRF mitigation
      ├── Input Validation: Client-side (UX) + Server-side (Security)
      ├── Database Schema: Normalization, indexes, constraints, migrations
      └── Frontend Stack Selection (Vanilla JS → Vue.js → JAMStack)
```

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
