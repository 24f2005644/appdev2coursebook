# 4. JSON (JavaScript Object Notation)

---

## 4.1 JSON Overview & Specifications

### What is JSON?
> **JSON** (JavaScript Object Notation) is a **lightweight, text-based data interchange format** used to represent structured data as a human-readable string.

It was originally derived from JavaScript object literal syntax, but it is now **language-independent** — virtually every programming language (Python, Java, Go, Rust, etc.) has libraries to read and write JSON.

**Primary use cases:**
- Sending data between a **client and server** over HTTP (REST APIs)
- **Configuration files** (`package.json`, `tsconfig.json`, `.eslintrc.json`)
- **Storing structured data** in files or databases (e.g., MongoDB documents)
- Inter-process communication

```
Client  ──── JSON string ────→  Server
Server  ──── JSON string ────→  Client
```

---

### What "Frozen Notation" Means
The name contains "Notation" — JSON is not a programming language, it has no logic, no functions, no variables. It is a **fixed, strict textual notation** for data.

Douglas Crockford (the creator of JSON) designed it with a deliberately minimal, strict specification. This strictness ensures interoperability across all languages and parsers.

---

### JSON Data Types
JSON supports exactly **6 value types** (no more, no less):

| JSON Type | Example | Notes |
|-----------|---------|-------|
| **String** | `"hello"` | **Must use double quotes** — single quotes not allowed |
| **Number** | `42`, `3.14`, `-7`, `1e5` | No distinction between int and float |
| **Boolean** | `true`, `false` | Lowercase only |
| **Null** | `null` | Lowercase only |
| **Array** | `[1, "two", true]` | Ordered list of any JSON values |
| **Object** | `{"key": "value"}` | Unordered key-value pairs |

> ⚠️ **Not in JSON**: `undefined`, functions, `Date`, `Symbol`, `Infinity`, `NaN`, `BigInt` — these JavaScript types have no JSON representation.

---

### JSON Syntax Rules (The "Frozen" Specification)

These rules are strict — violating any of them makes the JSON **invalid** and unparseable:

#### ① Keys Must Be Strings in Double Quotes
```json
// ✅ Valid
{ "name": "Alice" }

// ❌ Invalid — unquoted key
{ name: "Alice" }

// ❌ Invalid — single quotes
{ 'name': 'Alice' }
```

#### ② Strings Must Use Double Quotes
```json
"hello"   // ✅ valid
'hello'   // ❌ invalid
```

#### ③ No Trailing Commas
```json
// ✅ Valid
{
  "a": 1,
  "b": 2
}

// ❌ Invalid — trailing comma after last item
{
  "a": 1,
  "b": 2,
}
```

#### ④ No Comments
```json
// ❌ Invalid — JSON has NO comment syntax whatsoever
{
  // This is the user's name
  "name": "Alice"
}
```

> This is intentional. JSON is a data format, not a configuration language. (Tools like JSON5 or JSONC are extensions that allow comments, but they are not standard JSON.)

#### ⑤ No `undefined`, No Functions, No Special Values
```json
// ❌ Invalid — undefined is not a JSON type
{ "value": undefined }

// ❌ Invalid — functions are not data
{ "fn": function() {} }

// ❌ Invalid — Infinity and NaN are not JSON numbers
{ "x": Infinity, "y": NaN }
```

---

### A Valid JSON Example

```json
{
  "name": "Alice",
  "age": 30,
  "isStudent": false,
  "scores": [95, 87, 92],
  "address": {
    "city": "Delhi",
    "pin": "110001"
  },
  "nickname": null
}
```

All keys are double-quoted strings. Values use only the 6 allowed types. No trailing commas. No comments.

---

### JSON vs JavaScript Object Literal

| Feature | JSON | JS Object Literal |
|---------|------|-------------------|
| Key quotes | **Required** (double only) | Optional |
| String quotes | Double quotes only | Single or double |
| Trailing commas | ❌ Not allowed | ✅ Allowed |
| Comments | ❌ Not allowed | ✅ Allowed |
| Functions as values | ❌ Not allowed | ✅ Allowed |
| `undefined` | ❌ Not allowed | ✅ Allowed |
| `Date`, `RegExp` | ❌ Not allowed | ✅ Allowed |
| Valid in JS? | ✅ (valid JS expression) | ✅ |

> A valid JSON string is always valid JavaScript, but a JavaScript object literal is often **not** valid JSON.

---

## 4.2 The JSON API

JavaScript provides a global `JSON` object with exactly two methods for working with JSON. No import is needed — it is available everywhere (browser and Node.js).

```
JSON
 ├── JSON.stringify()  — JS value  →  JSON string   (Serialization)
 └── JSON.parse()      — JSON string  →  JS value   (Deserialization)
```

---

### `JSON.stringify()` — Serialization

> **Serialization**: Converting a JavaScript value into a JSON string so it can be stored or transmitted.

```javascript
const user = {
  name: "Alice",
  age: 30,
  isStudent: false,
  scores: [95, 87, 92],
  address: { city: "Delhi" }
};

const jsonString = JSON.stringify(user);
console.log(jsonString);
// '{"name":"Alice","age":30,"isStudent":false,"scores":[95,87,92],"address":{"city":"Delhi"}}'
```

The result is a **plain string** — safe to send over HTTP, write to a file, or store in a database.

---

#### Full Signature: `JSON.stringify(value, replacer, space)`

**`replacer`** (optional) — Filter or transform which properties to include:

```javascript
// As an array — whitelist of keys to include
JSON.stringify(user, ["name", "age"]);
// '{"name":"Alice","age":30}'

// As a function — custom transformation
JSON.stringify(user, (key, value) => {
  if (typeof value === 'number') return value * 2;  // double all numbers
  return value;
});
// '{"name":"Alice","age":60,"isStudent":false,"scores":[190,174,184],...}'
```

**`space`** (optional) — Pretty-print with indentation:

```javascript
console.log(JSON.stringify(user, null, 2));
```
```json
{
  "name": "Alice",
  "age": 30,
  "isStudent": false,
  "scores": [
    95,
    87,
    92
  ],
  "address": {
    "city": "Delhi"
  }
}
```

> `null, 2` means: no replacer, indent with 2 spaces. Extremely useful for logging and config files.

---

#### What Gets Lost During `stringify()`

Some JavaScript types have no JSON equivalent and are silently dropped or converted:

```javascript
const data = {
  name: "Alice",
  greet: function() { return "hi"; },  // ← function
  created: new Date(),                  // ← Date object
  score: undefined,                     // ← undefined
  symbol: Symbol("id"),                 // ← Symbol
  infinity: Infinity,                   // ← Infinity
  nan: NaN                              // ← NaN
};

JSON.stringify(data);
// '{"name":"Alice","created":"2026-09-04T...","infinity":null,"nan":null}'
// Note: function, undefined, symbol are OMITTED entirely
// Date is converted to its ISO string representation
// Infinity and NaN become null
```

> ⚠️ This "loss" is silent — no error is thrown. Always be aware of what types you're serializing.

---

#### Circular References — The `stringify()` Trap

```javascript
const a = {};
const b = { ref: a };
a.ref = b;  // a → b → a → b → ... (circular!)

JSON.stringify(a);  // ❌ TypeError: Converting circular structure to JSON
```

If you need to stringify objects with circular references, use a library like `flatted` or write a custom replacer.

---

### `JSON.parse()` — Deserialization

> **Deserialization**: Converting a JSON string back into a JavaScript value.

```javascript
const jsonString = '{"name":"Alice","age":30,"scores":[95,87,92]}';

const user = JSON.parse(jsonString);

console.log(user.name);    // "Alice"
console.log(user.age);     // 30
console.log(user.scores);  // [95, 87, 92]
console.log(typeof user);  // "object"
```

The output is a **live JavaScript object** — you can access properties, iterate arrays, call methods, etc.

---

#### Full Signature: `JSON.parse(text, reviver)`

**`reviver`** (optional) — Transform values during parsing:

```javascript
const json = '{"name":"Alice","birthdate":"1995-05-15"}';

const user = JSON.parse(json, (key, value) => {
  if (key === "birthdate") return new Date(value);  // convert string → Date
  return value;
});

console.log(user.birthdate instanceof Date);  // true ✅
console.log(user.birthdate.getFullYear());    // 1995
```

> The reviver is the counterpart to the `replacer` in `stringify()`. Use them together to serialize/deserialize types that JSON doesn't natively support (like `Date`).

---

#### Error Handling with `JSON.parse()`

`JSON.parse()` **throws a `SyntaxError`** if the string is not valid JSON. Always wrap it in `try...catch` when parsing untrusted input:

```javascript
function safeParse(str) {
  try {
    return JSON.parse(str);
  } catch (err) {
    console.error("Invalid JSON:", err.message);
    return null;
  }
}

safeParse('{"name":"Alice"}');    // { name: "Alice" } ✅
safeParse("not valid json");      // null, logs error ✅
safeParse('{name: "Alice"}');     // null — unquoted key is invalid JSON
safeParse("{'name': 'Alice'}");   // null — single quotes are invalid JSON
```

> **Never** call `JSON.parse()` without error handling on data coming from external sources (user input, API responses, files).

---

### The Full Round-Trip

```javascript
// 1. Start with a JS object
const original = { name: "Alice", scores: [95, 87] };

// 2. Serialize to JSON string (for transmission/storage)
const serialized = JSON.stringify(original);
// '{"name":"Alice","scores":[95,87]}'

// 3. Transmit / store the string...

// 4. Deserialize back to a JS object
const restored = JSON.parse(serialized);
// { name: "Alice", scores: [95, 87] }

// 5. The restored object is independent of the original
restored.name = "Bob";
console.log(original.name);  // "Alice" — unaffected
```

> This round-trip (`stringify` → transmit → `parse`) is the backbone of virtually every REST API call in web development.

---

### Practical: Fetching JSON from an API

```javascript
async function getUser(id) {
  const response = await fetch(`https://api.example.com/users/${id}`);
  
  // response.json() reads the response body and parses it automatically
  const user = await response.json();  // internally calls JSON.parse()
  
  console.log(user.name);
}
```

When **sending** JSON to a server:
```javascript
async function createUser(userData) {
  const response = await fetch('https://api.example.com/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(userData)  // serialize to JSON string for the request body
  });
  
  return response.json();
}
```

---

### Deep Clone Trick (Bonus)
A quick (but imperfect) way to deep-clone a plain object:

```javascript
const original = { a: 1, b: { c: 2 } };
const clone = JSON.parse(JSON.stringify(original));

clone.b.c = 99;
console.log(original.b.c);  // 2 — original unchanged ✅
```

> **Caveat**: This only works for JSON-serializable data. Functions, `Date` objects, `undefined`, and circular references are all lost or broken. For production, use `structuredClone()` (modern) or a library like Lodash's `_.cloneDeep()`.

---

## Summary

| Concept | Key Idea | Key Rule / Syntax |
|---------|----------|-------------------|
| **JSON** | Text format for structured data interchange | Language-independent, strict syntax |
| **6 Types** | string, number, boolean, null, array, object | No `undefined`, no functions, no `Date` |
| **Double quotes** | All strings and keys must use `"..."` | Single quotes = invalid JSON |
| **No trailing commas** | Last item in object/array has no trailing `,` | Strict parsing will throw |
| **No comments** | JSON has zero comment syntax | Use JSON5/JSONC for configs that need comments |
| **`JSON.stringify()`** | JS value → JSON string | `replacer` to filter, `space` to pretty-print |
| **`JSON.parse()`** | JSON string → JS value | `reviver` to transform; always wrap in `try...catch` |
| **Silent loss** | Functions, `undefined`, `Symbol` are silently dropped on stringify | Be explicit about what you're serializing |
| **Circular refs** | `stringify()` throws on circular structures | Use `flatted` or a custom replacer |
| **Round-trip** | `stringify` → transmit → `parse` | Core of every REST API interaction |
