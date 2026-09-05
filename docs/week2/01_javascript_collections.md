# 1. JavaScript Collections

---

## 1.1 Basic Arrays

### What is an Array?
> An **array** is an ordered collection of values, where each value can be of **any type**.

JavaScript arrays are **heterogeneous** — a single array can contain numbers, strings, booleans, objects, functions, and even other arrays simultaneously.

```javascript
const mixed = [42, "hello", true, { name: "Alice" }, () => "I'm a function", [1, 2, 3]];
```

This is unlike many statically-typed languages (e.g., Java) where all elements must be the same type.

---

### Element Access & Indexing
Arrays are **zero-indexed** — the first element is at index `0`.

```javascript
const fruits = ["apple", "banana", "cherry"];

fruits[0];  // "apple"
fruits[1];  // "banana"
fruits[2];  // "cherry"
fruits[3];  // undefined  ← out-of-bounds, no error
```

You can also write to any index:
```javascript
fruits[1] = "blueberry";  // replaces "banana"
fruits[5] = "date";       // sparse array — indices 3 and 4 are now "holes"
```

---

### Array `.length` Property & Dynamic Sizing
JavaScript arrays are **dynamic** — they grow and shrink automatically. The `.length` property always reflects the current array size.

```javascript
const arr = [10, 20, 30];
arr.length;    // 3

arr.push(40);  // append to end
arr.length;    // 4

arr.pop();     // remove from end → returns 40
arr.length;    // 3
```

> **Note**: `.length` is not read-only. Setting `arr.length = 0` empties the array.

Common mutation methods:

| Method | Action | Returns |
|--------|--------|---------|
| `push(x)` | Append to end | New length |
| `pop()` | Remove from end | Removed element |
| `shift()` | Remove from start | Removed element |
| `unshift(x)` | Prepend to start | New length |
| `splice(i, n)` | Remove `n` items at index `i` | Removed items |

---

### Array Holes (Sparse Arrays)
A **sparse array** has "holes" — indices that were never assigned a value.

```javascript
const sparse = [1, , , 4];   // indices 1 and 2 are holes
sparse.length;                // 4
sparse[1];                    // undefined
```

Holes are **different from `undefined`** — iteration methods like `forEach` skip holes, while explicitly set `undefined` values are processed.

```javascript
const a = [1, , 3];
a.forEach(x => console.log(x));  // logs: 1, 3  ← hole skipped!

const b = [1, undefined, 3];
b.forEach(x => console.log(x));  // logs: 1, undefined, 3
```

> ⚠️ Avoid creating sparse arrays intentionally — their behavior across iteration methods is inconsistent and error-prone.

---

## 1.2 Iteration & Iterable Protocol

### Definition
> **Iteration** is the process of accessing elements of a collection, one at a time, sequentially.

JavaScript has a formal **Iterable Protocol** — a standard interface that objects can implement to support sequential traversal.

---

### Two Core Concepts

| Concept | Description |
|---------|-------------|
| **Iterable** | An object whose contents can be accessed sequentially. It has a `[Symbol.iterator]()` method that returns an Iterator. |
| **Iterator** | A pointer/cursor to the current position in a sequence. It has a `.next()` method that returns `{ value, done }`. |

```javascript
const arr = [10, 20, 30];
const iterator = arr[Symbol.iterator]();  // get the iterator

iterator.next();  // { value: 10, done: false }
iterator.next();  // { value: 20, done: false }
iterator.next();  // { value: 30, done: false }
iterator.next();  // { value: undefined, done: true }  ← sequence exhausted
```

The `for...of` loop is syntactic sugar built on top of this protocol — it calls `.next()` under the hood.

```javascript
for (const item of arr) {
  console.log(item);  // 10, 20, 30
}
```

---

### Built-in Iterable Objects
These all implement the Iterable Protocol out of the box:

| Type | Example | Notes |
|------|---------|-------|
| `Array` | `[1, 2, 3]` | Most common iterable |
| `String` | `"hello"` | Iterates character by character |
| `Map` | `new Map()` | Iterates as `[key, value]` pairs |
| `Set` | `new Set()` | Iterates unique values in insertion order |
| Browser DOM | `document.querySelectorAll('p')` | NodeList is iterable |

```javascript
for (const char of "hello") {
  console.log(char);  // h, e, l, l, o
}
```

---

### Object Iteration Helpers
Plain objects (`{}`) do **not** implement the Iterable Protocol directly. Use these static methods instead:

```javascript
const person = { name: "Alice", age: 30, city: "Delhi" };

Object.keys(person);     // ["name", "age", "city"]       ← array of keys
Object.values(person);   // ["Alice", 30, "Delhi"]        ← array of values
Object.entries(person);  // [["name","Alice"], ["age",30], ["city","Delhi"]]  ← array of [key, value] pairs
```

These return arrays, so you can then iterate them with `for...of` or array methods:
```javascript
for (const [key, value] of Object.entries(person)) {
  console.log(`${key}: ${value}`);
}
// name: Alice
// age: 30
// city: Delhi
```

---

## 1.3 Iterations and Transformations (Functional Programming)

### Higher-Order Functions (HOFs)
> A **Higher-Order Function** is a function that **accepts another function as an argument** (and/or returns one).

This is the core of functional programming in JavaScript. Instead of writing explicit loops, you describe *what transformation* to apply to each element.

---

### Core Transformation Methods

All three of these are **non-mutating** — they return a new array and leave the original untouched.

#### `map(callback)` — Transform Every Element
> Apply a function to every element and return a **new array** of the results.

```javascript
const numbers = [1, 2, 3, 4];
const doubled = numbers.map(n => n * 2);
// doubled = [2, 4, 6, 8]
// numbers is unchanged: [1, 2, 3, 4]
```

#### `filter(callback)` — Keep Elements Matching a Condition
> Return a **new array** containing only the elements for which the callback returns `true`.

```javascript
const numbers = [1, 2, 3, 4, 5, 6];
const evens = numbers.filter(n => n % 2 === 0);
// evens = [2, 4, 6]
```

#### `find(callback)` — Find the First Matching Element
> Return the **first element** for which the callback returns `true`, or `undefined` if none match.

```javascript
const users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
  { id: 3, name: "Charlie" }
];
const user = users.find(u => u.id === 2);
// user = { id: 2, name: "Bob" }
```

> `find` returns the **element** itself, not a new array. Compare with `filter` which always returns an array.

---

### Callback Functions
> A **callback** is a function that you pass as an argument to another function, to be executed **at a later time** by that function.

```javascript
function greet(name, callback) {
  console.log("Hello, " + name);
  callback();             // execute the passed-in function
}

greet("Alice", () => {
  console.log("Callback executed!");
});
// Hello, Alice
// Callback executed!
```

In the context of array methods, the callback is called for each element automatically.

---

### Method Chaining
Because `map` and `filter` return arrays, you can **chain** them together:

```javascript
const result = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .filter(n => n % 2 === 0)   // [2, 4, 6, 8, 10]
  .map(n => n * n)             // [4, 16, 36, 64, 100]
  .filter(n => n > 20);        // [36, 64, 100]

console.log(result);  // [36, 64, 100]
```

> Chaining creates a **data pipeline** — a series of transformations applied in sequence. This is the essence of the functional style.

---

## 1.4 Other Collections

### `Map` — Proper Key-Value Dictionary
> A `Map` is an ordered collection of **key-value pairs** where **keys can be of any type** (not just strings like in plain objects).

```javascript
const map = new Map();

map.set("name", "Alice");    // string key
map.set(42, "answer");       // number key
map.set(true, "yes");        // boolean key
map.set({ id: 1 }, "obj");  // object key (by reference)

map.get("name");    // "Alice"
map.get(42);        // "answer"
map.size;           // 4  ← unlike plain objects, size is tracked
```

**`Map` vs Plain Object `{}`:**

| Feature | Plain Object `{}` | `Map` |
|---------|-------------------|-------|
| Key types | Only strings & Symbols | Any type |
| Order | Not guaranteed (in older JS) | Insertion order preserved |
| Size | `Object.keys(obj).length` | `map.size` (built-in) |
| Iteration | `Object.entries()` | `map.forEach()`, `for...of` |
| Performance | Optimized for static shapes | Optimized for frequent add/delete |

Maps are iterable — `for...of` gives `[key, value]` pairs:
```javascript
for (const [key, value] of map) {
  console.log(key, "→", value);
}
```

---

### `WeakMap` — Garbage-Collector-Friendly Map
> A `WeakMap` is like a `Map`, but its **keys must be objects** and they are held **weakly** — meaning if no other references to the key object exist, it can be garbage-collected automatically.

```javascript
let obj = { name: "Alice" };
const wm = new WeakMap();
wm.set(obj, "some data");

obj = null;  // The object can now be garbage-collected
             // The WeakMap entry is automatically cleaned up
```

**Use case**: Storing private metadata associated with objects (e.g., DOM element caches) without preventing those objects from being garbage-collected.

> `WeakMap` is **not iterable** and has no `.size` property — by design, you cannot enumerate its contents.

---

### `Set` — Collection of Unique Values
> A `Set` is an ordered collection that **only stores unique values** — duplicates are silently ignored.

```javascript
const set = new Set([1, 2, 3, 2, 1]);
console.log(set);   // Set { 1, 2, 3 }   ← duplicates removed
set.size;           // 3

set.add(4);         // Set { 1, 2, 3, 4 }
set.has(2);         // true
set.delete(2);      // removes 2
set.has(2);         // false
```

**Common use case — removing duplicates from an array:**
```javascript
const arr = [1, 2, 2, 3, 3, 3, 4];
const unique = [...new Set(arr)];
// [1, 2, 3, 4]
```

Sets are iterable and maintain insertion order:
```javascript
for (const val of set) {
  console.log(val);  // 1, 3, 4
}
```

---

## 1.5 Destructuring

> **Destructuring** is a JavaScript syntax that lets you **unpack values from arrays or properties from objects** into individual variables, in a single, concise expression.

### Array Destructuring
Pattern: left side mirrors the structure of the right side.

```javascript
const coords = [10, 20, 30];

const [x, y, z] = coords;
console.log(x);  // 10
console.log(y);  // 20
console.log(z);  // 30
```

**Skipping elements:**
```javascript
const [first, , third] = [1, 2, 3];  // skip index 1
// first = 1, third = 3
```

**Default values:**
```javascript
const [a = 0, b = 0] = [5];
// a = 5, b = 0 (default applied because index 1 is undefined)
```

**Swapping variables (classic trick):**
```javascript
let p = 1, q = 2;
[p, q] = [q, p];
// p = 2, q = 1
```

---

### Rest / Spread in Destructuring
The **rest element** (`...`) collects remaining elements into a new array:

```javascript
const [head, ...tail] = [1, 2, 3, 4, 5];
// head = 1
// tail = [2, 3, 4, 5]
```

As function parameters — **collecting** arguments:
```javascript
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4);  // 10
```

As function call arguments — **spreading** an array:
```javascript
const nums = [1, 2, 3];
Math.max(...nums);  // same as Math.max(1, 2, 3) → 3
```

---

### Object Destructuring
Extract named properties from an object into local variables:

```javascript
const user = { name: "Alice", age: 30, city: "Delhi" };

const { name, age } = user;
console.log(name);  // "Alice"
console.log(age);   // 30
```

**Renaming while destructuring:**
```javascript
const { name: userName, age: userAge } = user;
// userName = "Alice", userAge = 30
```

**Default values:**
```javascript
const { name, role = "viewer" } = { name: "Bob" };
// name = "Bob", role = "viewer"  (default applied)
```

**Nested destructuring:**
```javascript
const config = { db: { host: "localhost", port: 5432 } };
const { db: { host, port } } = config;
// host = "localhost", port = 5432
```

**In function parameters (very common in modern JS):**
```javascript
function greet({ name, age }) {
  console.log(`${name} is ${age}`);
}
greet({ name: "Alice", age: 30 });  // "Alice is 30"
```

---

## 1.6 Generators

### What is a Generator?
> A **generator** is a special function that can **pause its execution** and **yield values one at a time**, resuming on demand. It produces an **iterator** automatically.

Declared with `function*` syntax (the `*` is the distinguishing mark):

```javascript
function* count() {
  yield 1;
  yield 2;
  yield 3;
}
```

---

### How Generators Work
Calling a generator function does **not** execute its body — it returns a **Generator object** (which is an iterator).

```javascript
const gen = count();    // returns a Generator object, no code runs yet

gen.next();  // { value: 1, done: false }  ← runs until first yield
gen.next();  // { value: 2, done: false }  ← runs until second yield
gen.next();  // { value: 3, done: false }  ← runs until third yield
gen.next();  // { value: undefined, done: true }  ← function returned
```

Each `.next()` call **resumes** the generator from where it was last paused (after the previous `yield`).

---

### Lazy Evaluation
Generators implement **lazy evaluation** — values are computed only when requested, not all upfront.

```javascript
function* infiniteNumbers() {
  let n = 1;
  while (true) {
    yield n++;   // pauses after each yield, resumes when .next() is called
  }
}

const gen = infiniteNumbers();
gen.next().value;  // 1
gen.next().value;  // 2
gen.next().value;  // 3
// Can go on forever without consuming infinite memory
```

> ⚡ This is powerful — you can represent infinite sequences without pre-computing them.

---

### Using Generators with `for...of`
Since generators produce iterators (which implement the Iterable Protocol), you can use them with `for...of`:

```javascript
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

for (const num of range(1, 5)) {
  console.log(num);  // 1, 2, 3, 4, 5
}
```

---

### Computed / Dynamic Iterables
Generators let you build **custom, dynamically computed iterables** — sequences where each value is calculated on the fly:

```javascript
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
for (let i = 0; i < 7; i++) {
  console.log(fib.next().value);
}
// 0, 1, 1, 2, 3, 5, 8
```

---

## Summary

| Concept | Key Idea | Key Syntax |
|---------|----------|------------|
| **Arrays** | Ordered, dynamic, heterogeneous collection | `[]`, `.push()`, `.pop()` |
| **Iterable Protocol** | Standard interface for sequential access | `[Symbol.iterator]()`, `.next()`, `for...of` |
| **HOFs / Transforms** | Apply functions to collections functionally | `.map()`, `.filter()`, `.find()` |
| **Map** | Proper key-value dictionary (any key type) | `new Map()`, `.set()`, `.get()` |
| **WeakMap** | Weakly-held object keys, GC-friendly | `new WeakMap()` |
| **Set** | Collection of unique values | `new Set()`, `.add()`, `.has()` |
| **Destructuring** | Unpack arrays/objects into variables | `const [a, b] = arr`, `const {x} = obj` |
| **Generators** | Lazily produce values on demand | `function*`, `yield`, `.next()` |
