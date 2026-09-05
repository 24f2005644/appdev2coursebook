# Topic 1: Web API Design & REST Conventions

---

## 1.1 Purpose of an API & Developer-Centric Design

### What is a Web API?

An **API (Application Programming Interface)** is a set of rules and contracts that allows different software systems to communicate with each other. A **Web API** exposes these contracts over the internet using standard web protocols (primarily HTTP).

> Think of an API as a **restaurant menu** — it tells you exactly what you can order (endpoints), what ingredients you must provide (parameters), and what you'll get back (response format). You don't need to know how the kitchen works.

### Building Blocks for Applications

Modern applications are rarely monolithic. A Web API acts as:
- The **glue** between a frontend (web/mobile) and a backend database/service
- A **contract** enabling third parties to build products on top of your platform (e.g., Twitter's API, GitHub API, Stripe API)
- An **interoperability layer** connecting microservices to each other

```
Mobile App ──┐
Web App    ──┼──▶ REST API ──▶ Database / Services / Logic
3rd Party  ──┘
```

### Designing for Developers, Not End Users

Unlike a user interface (UI) designed for human visual interaction, an API is a **developer interface (DX — Developer Experience)**. Your consumers are engineers writing code, not people clicking buttons.

| UI Design | API Design |
|---|---|
| Intuitive visual navigation | Consistent, predictable endpoint structure |
| Helpful error messages in plain language | Machine-readable error codes + structured error payloads |
| Forgiving input (autocomplete, validation hints) | Strict, documented input contracts |
| Fast for humans to discover | Fast for developers to discover & integrate |

**Key implication**: A well-designed API should allow a developer to understand what it does just by reading the URL and HTTP method — without needing documentation for every call.

### Remote Procedure Call (RPC) vs. Web APIs

**RPC (Remote Procedure Call)** is an older paradigm where the client calls a named function on a remote server — as if calling a local function.

```
# RPC thinking: "I want to call a function on the server"
POST /getStudentById?id=42
POST /createNewStudent
POST /enrollStudentInCourse
```

**Web API (REST) thinking**: "I want to operate on a resource using a standardized action"

```
GET    /students/42          ← Read student 42
POST   /students             ← Create new student
POST   /courses/101/students ← Enroll student in course
```

The distinction matters because RPC leads to verb-heavy, inconsistent, and unpredictable endpoint structures that are hard to document and maintain.

### Reference: Apigee's *"Web API Design: The Missing Link"*

This influential guide (now published by Google Cloud) formalizes best practices for pragmatic REST API design. Key takeaways:
- APIs should be **developer-centric**
- Prefer **simplicity** over theoretical purity
- **Conventions over configurations** — follow what developers already expect

---

## 1.2 Data-Oriented API Modeling

Before writing a single endpoint, good API design starts with **modeling the domain**.

### Step 1: Identify Core Entities

Think of every significant "thing" (noun) in your domain:

| Domain | Core Entities |
|---|---|
| University System | Students, Courses, Grades, Professors, Departments |
| E-commerce | Products, Orders, Customers, Reviews, Payments |
| Social Network | Users, Posts, Comments, Likes, Follows |

Each entity will typically map to one or more API resources.

### Step 2: Define Core CRUD Actions

For each entity, identify which standard operations apply:

| Action | HTTP Method | Example |
|---|---|---|
| Create | `POST` | Add a new student |
| Read (one) | `GET` | Get student by ID |
| Read (many) | `GET` | List all students |
| Update (full) | `PUT` | Replace student record |
| Update (partial) | `PATCH` | Change student's email only |
| Delete | `DELETE` | Remove a student |

### Step 3: Define Summaries & Aggregations

Beyond raw CRUD, real APIs need to surface higher-level computed views:

- `GET /students` — paginated list of students
- `GET /students/top?limit=10` — top 10 students by GPA
- `GET /courses/123/gpa` — average GPA for a specific course
- `GET /departments/cs/stats` — enrollment and grade distribution

### Step 4: Complex / "Exotic" Queries

Some queries span multiple entities or require multi-attribute filtering:

- "Find all students enrolled in CS101 with GPA > 3.5 who haven't submitted assignment 3"
- These cannot be cleanly expressed as a single hierarchical URL
- Solution: Use **query string parameters** for filtering, sorting, pagination

```
GET /students?course=CS101&min_gpa=3.5&missing_assignment=3&page=2&limit=20
```

---

## 1.3 RPC-Style vs. Resource-Oriented REST Conventions

### The RPC Anti-Pattern

Without intentional design, APIs tend to grow into **RPC-style endpoint sprawl**:

```
/getListOfStudents
/getStudentById
/createNewStudent
/updateStudentEmail
/deleteStudent
/createStudentAndAddToCourse
/getCoursesForStudent
/getStudentsByDepartment
/getStudentsByGPARange
```

**Problems with this approach:**

1. **Cognitive overhead** — No discernible pattern; every endpoint must be memorized
2. **Documentation burden** — Every unique endpoint needs separate documentation
3. **Endpoint explosion** — Each new requirement spawns a new endpoint name
4. **Inconsistency** — Different developers name things differently (`getStudent` vs `fetchStudent` vs `retrieveStudent`)
5. **Not cacheable** — Verb-heavy POST-based endpoints cannot be cached by HTTP intermediaries

### The REST Convention Transition

REST maps **HTTP verbs** (GET, POST, PUT, PATCH, DELETE) onto **resource nouns** (URLs).

| RPC Style | REST Style | What Changed |
|---|---|---|
| `GET /getListOfStudents` | `GET /students` | URL is a noun; GET implies read |
| `POST /createNewStudent` | `POST /students` | POST on collection = create |
| `GET /getStudentById?id=42` | `GET /students/42` | ID in path, not query string |
| `POST /updateStudentEmail` | `PATCH /students/42` | PATCH on resource = partial update |
| `POST /deleteStudent?id=42` | `DELETE /students/42` | DELETE method is explicit |
| `POST /createStudentAndAddToCourse` | `POST /courses/101/students` | Relationship expressed in URL hierarchy |

> **The mental model shift**: You don't call functions anymore. You **operate on resources** using a small, fixed vocabulary of verbs (HTTP methods).

---

## 1.4 URL Naming Conventions & Structure

### Nouns in URLs, Verbs in HTTP Methods

The HTTP method *is* the verb. The URL should only contain nouns (resources).

```
Bad:
GET  /getAllStudents
POST /createStudent
POST /deleteStudent/42

Good:
GET    /students
POST   /students
DELETE /students/42
```

**Collections vs. Individual Resources:**

| Resource | URL Pattern |
|---|---|
| Collection of all students | `/students` |
| A single student | `/students/42` |
| A student's courses | `/students/42/courses` |
| A specific course enrollment | `/students/42/courses/CS101` |

### API Discoverability from Root (`/`)

A well-designed API should be **crawlable** — a developer hitting `GET /` should receive a response that lists the top-level resources available:

```json
GET /

{
  "students_url": "/students",
  "courses_url": "/courses",
  "departments_url": "/departments",
  "docs_url": "https://api.university.edu/docs"
}
```

This mirrors how browsing the web works — following links from a root. This property is part of the **HATEOAS** principle (covered in 1.8).

### Permalinks: Stable, Unique Identifiers

Every individual resource must have a **stable, unique URL** called a **permalink**:

| ID Type | Pros | Cons |
|---|---|---|
| **Integer ID** (`42`) | Simple, fast DB lookup | Exposes internal IDs, enumerable (security risk) |
| **UUID** (`a1b2-...`) | Non-guessable, globally unique | Long, ugly in URLs |
| **Slug** (`john-doe`) | Human-readable, SEO-friendly | May change (name updates break permalinks) |

**Best practice**: Use database-assigned IDs or UUIDs for stability. If you need human-readable, support *both* (`/students/42` and `/students/john-doe` mapping to the same resource).

---

## 1.5 Hierarchical vs. Query URLs

### Hierarchical Routes (Path Segments)

Used when there is a **clear parent-child relationship** between resources:

```
/students/42/courses           ← Courses enrolled by student 42
/courses/CS101/students        ← Students enrolled in CS101
/departments/cs/courses        ← Courses in the CS department
/courses/CS101/assignments/3   ← Assignment 3 of course CS101
```

**Use when**: The relationship is fixed, structural, and meaningful in isolation.

### Query String Parameters

Used for **filtering, sorting, pagination, and complex lookups** that don't imply a resource hierarchy:

```
/students?department=cs&min_gpa=3.5&sort=gpa_desc&page=2&limit=20
/courses?semester=fall2024&level=graduate&open=true
/students?search=john&enrolled_after=2023-01-01
```

**Use when**: You need flexible, multi-dimensional filtering on a collection.

### Developer Experience Considerations

Avoid creating "fake" hierarchical paths from query parameters:

```
Avoid:
/course-123-students         ← Hyphenated compound (not crawlable, ugly)
/students-in-cs101           ← Baked-in filter (creates endpoint proliferation)

Prefer:
/courses/123/students        ← True hierarchy (crawlable, meaningful)
/students?course=123         ← Query filter (flexible, composable)
```

---

## 1.6 HTTP Verbs & Semantics

HTTP defines a fixed vocabulary of methods, each with precise semantics. Using them correctly unlocks HTTP-level benefits like caching and idempotency.

### GET — Read

- **Safe**: Reading never modifies server state
- **Idempotent**: Calling it 1x or 100x has the same effect
- **Cacheable**: HTTP proxies, CDNs, and browsers cache GET responses by URL
- Never include a body in a GET request

```http
GET /students/42 HTTP/1.1
Host: api.university.edu
```

### POST — Create / Complex Actions

- **Not safe**: Creates or modifies state
- **Not idempotent**: Calling it twice creates two resources
- **Not cacheable** by default
- Also used for **complex queries** where GET's URL length limit is a problem

```http
POST /students HTTP/1.1
Content-Type: application/json

{ "name": "Jane Doe", "email": "jane@uni.edu" }
```

### PUT — Full Replacement

- **Idempotent**: Calling it multiple times always results in the same state
- **Replaces the entire resource** — missing fields are treated as deleted/null
- Use when the client is sending the **complete** new state of a resource

```http
PUT /students/42 HTTP/1.1
Content-Type: application/json

{ "name": "Jane Doe", "email": "jane@uni.edu", "gpa": 3.8, "department": "CS" }
```

### PATCH — Partial Update (Preferred for Updates)

- **Partially updates** a resource — only the fields sent are changed
- More network-efficient (smaller payloads)
- **Preferred over PUT** for most update operations in practice

```http
PATCH /students/42 HTTP/1.1
Content-Type: application/json

{ "email": "newemail@uni.edu" }
```

> Only `email` is updated; all other fields remain unchanged.

### DELETE — Remove

- **Idempotent**: Deleting a resource that's already gone still returns success (or 404)

```http
DELETE /students/42 HTTP/1.1
```

### Comparison Table

| Method | Safe | Idempotent | Cacheable | Body | Use For |
|---|---|---|---|---|---|
| `GET` | Yes | Yes | Yes | No | Read resource(s) |
| `POST` | No | No | No | Yes | Create, complex queries |
| `PUT` | No | Yes | No | Yes | Full replacement |
| `PATCH` | No | Usually | No | Yes | Partial update |
| `DELETE` | No | Yes | No | No | Remove resource |

### Conventions vs. Strict Rules

REST is an **architectural style**, not a strict protocol. Pragmatic deviations are acceptable:
- Some APIs use `POST` for all updates (legacy systems, simplicity)
- What matters most: **be consistent within your own API**

---

## 1.7 Data Serialization & Output Formats

### XML (eXtensible Markup Language)

An older, more verbose format dominant in the SOAP/enterprise era:

```xml
<student>
  <id>42</id>
  <name>Jane Doe</name>
  <courses>
    <course>CS101</course>
    <course>MATH201</course>
  </courses>
</student>
```

**Pros**: Rich type system, schemas (XSD), namespaces, excellent for documents
**Cons**: Verbose, harder to parse in JS, larger payload sizes, overkill for most APIs

### JSON (JavaScript Object Notation)

The modern standard for web APIs:

```json
{
  "id": 42,
  "name": "Jane Doe",
  "courses": ["CS101", "MATH201"]
}
```

**Pros**: Concise, native to JavaScript, human-readable, easy to parse in virtually every language
**Cons**: Limited primitive types (no Date, no distinction between int/float), no native schema enforcement

### JSON's Type System Limitations

JSON only supports: `string`, `number`, `boolean`, `null`, `array`, `object`

It does **not** natively support:
- Dates → use ISO 8601 strings: `"2024-01-15T09:00:00Z"`
- Integers vs. floats → both are just `number`
- Binary data → use Base64 encoding

### JSON + Extensions as the Modern Standard

| Extension | Purpose |
|---|---|
| **JSON:API** | Specification for structuring resources, relationships, and errors |
| **JSON Schema** | Vocabulary for validating JSON structure (like XSD for JSON) |
| Media type: `application/json` | Standard MIME type for JSON payloads |

---

## 1.8 Hypermedia & Included Links (HATEOAS)

### The Problem with Isolated JSON Payloads

A basic API response gives you the data, but nothing else:

```json
{
  "id": 42,
  "name": "Jane Doe",
  "department_id": 5
}
```

To find the department's details, a developer must separately consult documentation to know the URL is `/departments/5`. The API response itself provides no navigation.

### HATEOAS — Hypermedia as the Engine of Application State

HATEOAS is a REST principle: responses should include **links to related resources and available actions**, enabling clients to navigate the API like browsing the web.

```json
{
  "id": 42,
  "name": "Jane Doe",
  "department_id": 5,
  "_links": {
    "self":       { "href": "/students/42" },
    "department": { "href": "/departments/5" },
    "courses":    { "href": "/students/42/courses" },
    "update":     { "href": "/students/42", "method": "PATCH" },
    "delete":     { "href": "/students/42", "method": "DELETE" }
  }
}
```

### Real-World Case Study: GitHub REST API

GitHub's API includes links to all related endpoints in every response:

```json
{
  "id": 1296269,
  "name": "Hello-World",
  "full_name": "octocat/Hello-World",
  "url": "https://api.github.com/repos/octocat/Hello-World",
  "forks_url": "https://api.github.com/repos/octocat/Hello-World/forks",
  "commits_url": "https://api.github.com/repos/octocat/Hello-World/commits{/sha}",
  "issues_url": "https://api.github.com/repos/octocat/Hello-World/issues{/number}",
  "pulls_url": "https://api.github.com/repos/octocat/Hello-World/pulls{/number}"
}
```

The client receives the data *and* the URLs for every related action — no external documentation needed to discover next steps.

---

## 1.9 API Authentication & Security

### Token-Based Authentication

The most common pattern for stateless API authentication:

1. Client sends credentials to an auth endpoint
2. Server verifies credentials and returns a **token**
3. Client stores the token and sends it with every request (`Authorization` header)
4. Server validates the token on each request — no session state needed

```
Client                          Server
  │  POST /auth/login              │
  │  { email, password }  ────────▶│
  │◀──────────────── 200 OK        │
  │  { token: "eyJhbGc..." }       │
  │                                │
  │  GET /students                 │
  │  Authorization: Bearer eyJ... ─▶│  Verify token → allow
  │◀──────────────── 200 OK        │
  │  { students: [...] }           │
```

### OAuth2 — Delegated Authorization

**OAuth2** is an industry-standard protocol for allowing a third party to access resources **on behalf of a user — without sharing the user's password**.

**Classic use case**: "Login with Google", "Connect with GitHub"

**Key roles in OAuth2**:
- **Resource Owner**: The user
- **Client**: Your application
- **Authorization Server**: Issues tokens (e.g., Google Auth)
- **Resource Server**: Holds the protected data (e.g., Google Drive API)

```
User  ──▶  Your App  ──▶  Google Auth Server
                │              │
                │  Auth Code   │
                ◀──────────────│
                │  Exchange code for Access Token
                ──────────────▶│
                ◀── Access Token ─│
                │
                │  Request with Access Token
                ──────────────────────────────▶  Google API (Resource Server)
```

### JWT — JSON Web Tokens

A **JWT (JSON Web Token)** is a compact, self-contained token. Structure: `header.payload.signature`

**Decoded Payload (Claims)**:
```json
{
  "sub": "42",
  "name": "Jane Doe",
  "email": "jane@uni.edu",
  "role": "student",
  "iat": 1600000000,
  "exp": 1600086400
}
```

**Why JWTs are useful**:
- **Stateless**: Server verifies the signature cryptographically — no session DB lookup
- **Self-describing**: Payload carries user info, so no extra DB lookup needed per request
- **Expiry built-in**: `exp` claim enforces token lifetime

> Standard JWT is **signed** (tamper-proof) but **not encrypted** — payload is Base64-readable. Never store secrets in JWT payload.

### Why Use Industry-Standard Protocols?

- **Security is hard**: Rolling your own auth is a common source of critical vulnerabilities
- **Ecosystem support**: JWTs and OAuth2 have battle-tested libraries in every language
- **Interoperability**: Standard tokens work across services and enable SSO (Single Sign-On)

---

## 1.10 Principles & Philosophy of Good API Design

### API Design as an Experience-Driven Craft

A great API is not just technically correct — it is a **pleasurable developer experience**. The quality of an API directly determines how quickly and correctly developers can build on top of it.

**Signs of a well-designed API:**
- A developer can guess the URL of a new resource without reading docs
- Error messages clearly explain what went wrong and how to fix it
- The API behaves consistently — no surprises between endpoints
- Common tasks require minimal lines of code to accomplish

### Balancing Conventions with Practical Trade-offs

Strict theoretical REST purity is rarely the right goal:

| Theoretical REST Purist | Pragmatic Real-World API Designer |
|---|---|
| Every resource must have a unique URL | Use query params for complex searches |
| Only use HTTP methods per spec | Use POST for GraphQL queries (body too long for GET) |
| HATEOAS links on every response | Include links on complex resources; skip on trivial ones |
| No verbs in URLs ever | `/search`, `/login`, `/logout` are acceptable |

> The **goal** is an API that is **consistent, predictable, and pleasant to use** — not one that slavishly follows a theoretical model at the cost of developer experience.

### Quick-Reference Summary

| Principle | Rule of Thumb |
|---|---|
| Use nouns in URLs | `/students`, not `/getStudents` |
| Use HTTP verbs correctly | `GET`=read, `POST`=create, `PATCH`=partial update, `DELETE`=remove |
| Hierarchical URLs for relationships | `/students/42/courses` |
| Query params for filtering | `?min_gpa=3.5&sort=name` |
| Return JSON by default | `Content-Type: application/json` |
| Include navigational links | `_links.self`, `_links.related` |
| Use standard auth protocols | OAuth2 + JWT |
| Be consistent | Same conventions across all endpoints |
| Design for developers | Optimize for DX, not theoretical purity |
