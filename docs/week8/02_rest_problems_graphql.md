# Topic 2: Problems with REST & The Rise of GraphQL

---

## 2.1 Limitations and Friction Points in REST

### REST as a Style, Not a Spec

REST (Representational State Transfer) was defined by Roy Fielding in his 2000 doctoral dissertation as a set of **architectural constraints** — not a protocol or a standard. This means:

- There is no official REST specification to validate against
- Different teams implement "REST" differently in practice
- What most people call REST is actually **pragmatic HTTP APIs loosely inspired by REST principles**

This gap between theoretical REST and real-world REST is the root source of its friction points.

---

### The "Chatty Network" Problem (Cascading Round Trips)

One of the most painful real-world limitations of REST is the **N+1 request problem** — a single page view requiring many sequential API calls.

**Scenario**: Build a social media profile page that shows:
- User's name and avatar
- Their last 5 posts
- Comments on each post
- Likes count on each comment

**REST approach:**

```
Round Trip 1:  GET /users/42
               → { id: 42, name: "Jane", avatar_url: "...", post_ids: [1,2,3,4,5] }

Round Trip 2:  GET /posts/1
Round Trip 3:  GET /posts/2
Round Trip 4:  GET /posts/3
Round Trip 5:  GET /posts/4
Round Trip 6:  GET /posts/5

Round Trip 7:  GET /posts/1/comments
Round Trip 8:  GET /posts/2/comments
...and so on for each post
```

**Result**: A single profile page might require **10–20+ sequential HTTP round trips**, each with:
- DNS lookup + TCP handshake + TLS handshake overhead
- Server processing time
- Network latency (especially painful on mobile / high-latency connections)

The sequential nature makes this even worse — you can't fetch comments until you have post IDs, which you can't get until you have the user.

---

### Over-Fetching: Getting Too Much Data

REST endpoints return **fixed response shapes** — the server decides what's in the response, not the client.

**Example**: Fetching a user to display only their name in a navigation bar

```http
GET /users/42
```

```json
{
  "id": 42,
  "name": "Jane Doe",
  "email": "jane@example.com",
  "bio": "Software engineer...",
  "avatar_url": "https://cdn.example.com/avatars/jane.jpg",
  "created_at": "2021-03-15T10:00:00Z",
  "last_login": "2024-01-10T14:32:00Z",
  "preferences": { "theme": "dark", "notifications": true, ... },
  "social_links": { "twitter": "...", "github": "...", ... },
  "follower_count": 1240,
  "following_count": 389
}
```

The client only needed `name`, but received a payload with 15+ fields. At scale:
- **Wasted bandwidth** (mobile data costs, slow connections)
- **Larger payloads** that take longer to serialize/deserialize
- **Increased memory usage** on both client and server

---

### Under-Fetching: Not Getting Enough Data

The opposite problem — a single endpoint doesn't return everything the client needs, forcing additional requests.

**Example**: Displaying a blog post with author info

```http
GET /posts/99
→ { id: 99, title: "...", body: "...", author_id: 42 }  ← Only has author_id!

GET /users/42
→ { ... }  ← Must make a second request just to get the author's name
```

The client gets `author_id` but not the author's data — it has to **make another round trip** just because the endpoint doesn't include related data by default.

---

### Rigid Endpoint Structure

REST APIs have **fixed endpoints with fixed response shapes**. Adding a new client with different data needs means:
- Creating new endpoints (`/posts/summary`, `/posts/full`, `/posts/mobile`)
- Adding query parameters to control fields (`?fields=id,title,author`)
- Creating versioned APIs (`/v2/posts`)

All of these are workarounds for the fundamental rigidity of REST endpoint design.

---

### Inability to Express Arbitrary Queries

REST has no native way to express:
- "Give me all users who joined in 2023 AND have posted at least 5 times AND follow more than 100 people"
- Complex filtering across multiple related entities in a single request
- Recursive relationships (e.g., comments on comments on comments)

These require either custom endpoints, complex query strings, or multiple requests.

---

## 2.2 Motivation Behind GraphQL

GraphQL was developed internally at **Facebook (Meta) around 2012** and open-sourced in **2015**. The team was rebuilding the Facebook mobile app and ran directly into REST's limitations at scale.

### The Core Problem Facebook Faced

The Facebook News Feed is an extremely complex composite view:
- Posts from friends, pages, groups
- Each post: author info, media, like/comment counts, share counts
- Comments: author, text, nested replies
- Reactions: types, counts, user reactions

Under REST, rendering the News Feed required **dozens of round trips** with massively over-fetched responses — catastrophic for mobile users on slow networks.

### The Solution: A Query Language for APIs

GraphQL's core insight: **flip the data-fetching model**

```
REST:  Server defines what data you get (fixed endpoints, fixed shapes)
GraphQL:  Client declares exactly what data it needs (query language)
```

Instead of the server deciding the response shape, the **client writes a query** describing exactly what fields and relationships it needs — and the server returns exactly that, nothing more, nothing less.

### Declarative Programming Applied to Data Fetching

GraphQL applies the same **declarative programming** paradigm that made React revolutionary for UIs:

```
Imperative (REST):
  1. Fetch user
  2. Extract post IDs
  3. For each post ID, fetch post
  4. For each post, fetch comments
  5. Assemble the data
  → You describe HOW to get the data

Declarative (GraphQL):
  "I need user 42 with their last 5 posts, each with their top 3 comments"
  → You describe WHAT data you want; the server figures out HOW
```

This is the same shift as:
- SQL vs. iterating over rows manually
- React's JSX vs. manually manipulating the DOM

---

## 2.3 Architecture & Fundamentals of GraphQL

### GraphQL as a Specification, Not an Implementation

GraphQL is a **specification** (defined by the GraphQL Foundation) that describes:
- The query language syntax
- The type system
- The execution semantics
- The introspection system

Implementations exist in every major language: `graphql-js`, `Apollo Server` (Node.js), `Strawberry` / `Graphene` (Python), `gql` (Python client), etc.

### A Single Endpoint

Unlike REST (many endpoints), GraphQL exposes **one endpoint**:

```
REST:
  GET  /users/42
  GET  /users/42/posts
  GET  /posts/99/comments
  POST /users
  ...dozens more

GraphQL:
  POST /graphql   ← everything goes through one endpoint
```

All queries, mutations, and subscriptions are sent as HTTP `POST` requests to `/graphql` with the query in the request body.

### How a GraphQL Request Works

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{
  "query": "{ user(id: 42) { name email posts { title } } }"
}
```

**Response:**

```json
{
  "data": {
    "user": {
      "name": "Jane Doe",
      "email": "jane@example.com",
      "posts": [
        { "title": "First Post" },
        { "title": "Second Post" }
      ]
    }
  }
}
```

The client asked for `name`, `email`, and each post's `title` — and that's **exactly** what came back. Nothing more.

### The Server-Side Translation Engine (Resolvers)

When the GraphQL server receives a query, it translates it into actual data-fetching logic via **resolvers** — functions that fetch data for each field:

```
Query: { user(id: 42) { name posts { title } } }

GraphQL Engine
  │
  ├── resolver: user(id: 42) → SELECT * FROM users WHERE id = 42
  └── resolver: user.posts   → SELECT * FROM posts WHERE user_id = 42
      └── resolver: post.title → post.title (already in data)
```

Each field in the schema can have its own resolver — which can call a database, REST API, microservice, cache, or any other data source.

---

## 2.4 Type System & Schema Relations

### Strongly Typed Schema

Every GraphQL API is described by a **schema** written in **SDL (Schema Definition Language)**:

```graphql
type Student {
  id: ID!
  name: String!
  email: String!
  gpa: Float
  courses: [Course!]!
}

type Course {
  id: ID!
  title: String!
  code: String!
  students: [Student!]!
  professor: Professor
}

type Professor {
  id: ID!
  name: String!
  department: String!
}

type Query {
  student(id: ID!): Student
  students: [Student!]!
  course(code: String!): Course
}
```

### Scalar Types (Primitives)

| GraphQL Type | Description |
|---|---|
| `String` | UTF-8 text |
| `Int` | 32-bit signed integer |
| `Float` | Double-precision floating point |
| `Boolean` | `true` or `false` |
| `ID` | Unique identifier (serialized as String) |

You can also define **custom scalar types** (e.g., `DateTime`, `URL`, `JSON`).

### Non-Null and List Modifiers

```graphql
name: String      # nullable String (can be null)
name: String!     # non-null String (must have a value)
courses: [Course] # nullable list of nullable Courses
courses: [Course!]! # non-null list of non-null Courses
```

### Compile-Time / Query-Time Static Error Checking

Because the schema is strongly typed, GraphQL can **validate queries before execution**:

```graphql
# This query will be REJECTED before hitting any database:
{
  student(id: "42") {
    fullName    # Error: field 'fullName' doesn't exist (it's 'name')
    gpa
    salary      # Error: field 'salary' doesn't exist on Student
  }
}
```

Errors are caught **statically** — a huge improvement over REST where you discover shape mismatches at runtime.

### Explicit Entity Relations in the Schema

Relations between types are first-class citizens:

```graphql
type Student {
  courses: [Course!]!   # Student has many Courses
}

type Course {
  students: [Student!]! # Course has many Students
  professor: Professor  # Course has one Professor
}
```

This means the client can traverse relationships in a single query:

```graphql
{
  student(id: 42) {
    name
    courses {
      title
      professor {
        name
        department
      }
    }
  }
}
```

**This is a single request** that would require 3–4 REST round trips.

---

## 2.5 Versionless API Evolution

### The REST Versioning Problem

When a REST API changes (new fields, renamed fields, removed fields), clients may break. The common solution is **API versioning**:

```
/v1/students   ← original clients continue using this
/v2/students   ← new clients get new schema
/v3/students   ← further changes...
```

**Problems with URL versioning:**
- Maintenance burden: support multiple versions simultaneously
- Clients must explicitly migrate to new versions
- Documentation fragmentation
- Server complexity

### GraphQL's Additive Evolution Model

GraphQL schemas are designed to **evolve without breaking existing clients**:

**Adding fields** — always safe, existing queries are unaffected:
```graphql
# Before
type Student {
  id: ID!
  name: String!
}

# After (adding graduationYear — existing clients just don't request it)
type Student {
  id: ID!
  name: String!
  graduationYear: Int   # new field — completely safe to add
}
```

**Removing / renaming fields** — use `@deprecated` instead of deleting:
```graphql
type Student {
  id: ID!
  name: String!
  full_name: String @deprecated(reason: "Use 'name' instead")
}
```

The `@deprecated` annotation:
- Flags the field in documentation and tooling
- Still works for existing clients (no breakage)
- Signals to clients they should migrate
- Can be removed only after monitoring shows zero usage

This means **no `/v2`, no `/v3`** — the schema evolves continuously while maintaining backward compatibility.

---

## 2.6 Modifying Data with Mutations

In GraphQL, **queries** are read operations, while **mutations** are write operations (Create, Update, Delete).

### Mutation Syntax

```graphql
mutation {
  createStudent(name: "John Smith", email: "john@uni.edu") {
    id
    name
    email
  }
}
```

Response:
```json
{
  "data": {
    "createStudent": {
      "id": "43",
      "name": "John Smith",
      "email": "john@uni.edu"
    }
  }
}
```

The mutation returns the **newly created/updated resource** — the client gets fresh, confirmed data immediately without a follow-up fetch.

### Mutation Examples

```graphql
# Create
mutation {
  createStudent(name: "John", email: "john@uni.edu") {
    id name
  }
}

# Update
mutation {
  updateStudent(id: "42", email: "newemail@uni.edu") {
    id email updatedAt
  }
}

# Delete
mutation {
  deleteStudent(id: "42") {
    success
    message
  }
}

# Enroll in a course
mutation {
  enrollStudent(studentId: "42", courseCode: "CS101") {
    student { name }
    course { title }
    enrolledAt
  }
}
```

### Why Separate Query and Mutation?

The distinction is more than semantic — it maps to HTTP semantics:
- **Queries** are like `GET` — safe, can be cached, can be executed in parallel
- **Mutations** are like `POST/PATCH/DELETE` — have side effects, must be executed serially

GraphQL enforces this: **multiple mutations in one request execute sequentially**, while multiple queries can be parallelized.

---

## 2.7 Tooling & Ecosystem

### Apollo Server

**Apollo** is the most widely-used GraphQL implementation ecosystem:

**Apollo Server** (backend):
- Builds a production-ready GraphQL API on top of Node.js
- Supports **federated architecture** — split a large schema across multiple services and stitch them together
- Handles authentication, caching, error formatting, subscriptions
- Works with any data source: SQL, NoSQL, REST APIs, microservices

```javascript
const { ApolloServer, gql } = require('apollo-server');

const typeDefs = gql`
  type Student {
    id: ID!
    name: String!
    courses: [Course!]!
  }
  type Query {
    student(id: ID!): Student
  }
`;

const resolvers = {
  Query: {
    student: (_, { id }) => db.students.findById(id),
  },
  Student: {
    courses: (student) => db.courses.findByStudentId(student.id),
  },
};

const server = new ApolloServer({ typeDefs, resolvers });
server.listen();
```

### Custom Resolvers

Resolvers are the **bridge between the schema and the actual data**. Each resolver can independently fetch from:
- A PostgreSQL database
- A MongoDB collection
- A REST API (even a third-party one!)
- An in-memory cache
- Another microservice

This is why GraphQL is often used as an **API gateway** — it aggregates data from multiple backend services into a unified graph.

```javascript
const resolvers = {
  Student: {
    // This field fetches from a REST API, not the DB!
    courses: async (student) => {
      const response = await fetch(`https://courses-service.internal/students/${student.id}/courses`);
      return response.json();
    }
  }
};
```

### Interactive GraphQL Explorers

One of GraphQL's killer features is **built-in introspection** — the API can describe itself. This powers interactive tools:

| Tool | Description |
|---|---|
| **GraphiQL** | Browser-based IDE for writing & testing queries, built-in autocomplete |
| **Apollo Studio / Sandbox** | Full-featured explorer with schema visualization, query history |
| **GitHub GraphQL Explorer** | GitHub's public GraphQL API explorer — live, authenticated |
| **Postman** | Also supports GraphQL queries natively |

Because the schema is introspectable, these tools provide **autocomplete and inline documentation** — the API is self-documenting.

---

## 2.8 Server Complexity & Trade-off Analysis

GraphQL is not a silver bullet. It shifts complexity from the client to the server.

### The N+1 Query Problem (Server-Side)

GraphQL's flexible querying can cause a **database N+1 problem** on the server:

```graphql
{
  students {        # 1 query: SELECT * FROM students → returns 100 students
    courses {       # 100 queries: SELECT * FROM courses WHERE student_id = ?
      title         # (one query per student!) = 101 total queries
    }
  }
}
```

**Solution**: The **DataLoader** pattern — batch and cache DB queries:

```javascript
const DataLoader = require('dataloader');

const courseLoader = new DataLoader(async (studentIds) => {
  // Batch: SELECT * FROM courses WHERE student_id IN (1, 2, 3, ..., 100)
  // Just 1 query instead of 100!
  const courses = await db.courses.findByStudentIds(studentIds);
  return studentIds.map(id => courses.filter(c => c.studentId === id));
});
```

### Caching Challenges

REST APIs benefit naturally from HTTP caching — `GET /students/42` can be cached by URL at CDN and browser level.

GraphQL uses `POST` requests with query bodies — **HTTP-level caching doesn't work** out of the box. Solutions:
- **Persisted queries**: Pre-register queries server-side, then reference by hash (GET-cacheable)
- **Apollo Client** caches normalized data in the client
- **Server-side caching**: Cache resolver results by field/argument combinations

### Query Complexity & Depth Limiting

Malicious or accidental deeply nested queries can overload the server:

```graphql
# Potentially catastrophic query:
{
  students {
    courses {
      students {
        courses {
          students {
            ...
          }
        }
      }
    }
  }
}
```

GraphQL servers should implement:
- **Depth limiting**: Reject queries deeper than N levels
- **Query complexity scoring**: Assign a cost to each field and reject queries above a threshold
- **Rate limiting**: Cap queries per API key per time window

### REST vs. GraphQL: When to Use Which

| Aspect | REST | GraphQL |
|---|---|---|
| **Best for** | Simple CRUD, public APIs, file uploads | Complex data requirements, multiple clients with different needs |
| **Caching** | Excellent (HTTP-native) | Requires extra effort (persisted queries, Apollo Client) |
| **Learning curve** | Low | Higher (schema, resolvers, DataLoader, etc.) |
| **Tooling** | Mature, widespread | Rapidly maturing (Apollo, Relay) |
| **Over/Under-fetching** | Common problem | Solved by design |
| **Versioning** | Needs `/v1`, `/v2` | Versionless (additive schema changes) |
| **Server complexity** | Simple | Higher (N+1, depth limiting, complexity analysis) |
| **Introspection/Docs** | OpenAPI/Swagger (external) | Built-in, self-documenting |
| **Mobile performance** | Can be poor (chatty, over-fetch) | Excellent (precise queries, minimal data) |
