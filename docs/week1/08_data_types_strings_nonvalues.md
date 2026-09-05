# Topic 8: Data Types, Strings & Non-Values

> **MAD-II Week 1 | Detailed Notes**

---

## Overview

Every value in JavaScript has a **type** — a classification that determines what operations can be performed on it and how it behaves in expressions. JavaScript's type system is **dynamic** (types are associated with values, not variables) and has some genuinely unusual characteristics that distinguish it from most other languages.

The type system divides into two fundamental categories:
- **Primitives** — simple, immutable values
- **Objects** — complex, mutable compound structures (including functions)

Understanding this division — and its implications — is essential for writing correct JavaScript.

---

## 8.1 Primitive Types

JavaScript has **7 primitive types**. Primitives are:
- **Immutable** — the value itself cannot be changed (though a variable holding it can be reassigned)
- **Compared by value** — two primitives are equal if their values are equal
- **Stored by value** — when assigned to a variable or passed to a function, a copy is made

### The 7 Primitive Types

---

#### 1. Number

JavaScript has only **one numeric type** — it handles both integers and floating-point numbers using the **IEEE 754 double-precision 64-bit floating-point format**.

```javascript
// Both integers and floats are the same type:
typeof 42        // "number"
typeof 3.14      // "number"
typeof -100      // "number"

let x = 42;
let y = 3.14;
let z = -100;
```

**Safe integer range:**
- JavaScript can represent integers exactly in the range: **-(2⁵³ - 1) to (2⁵³ - 1)**
- `Number.MAX_SAFE_INTEGER` = 9,007,199,254,740,991 (about 9 quadrillion)
- `Number.MIN_SAFE_INTEGER` = -9,007,199,254,740,991
- Integers outside this range lose precision (use `BigInt` instead)

**Special numeric values:**
```javascript
Infinity           // 1 / 0 → positive infinity
-Infinity          // -1 / 0 → negative infinity
NaN                // "Not a Number" — result of invalid numeric operations

// NaN is the only value that is not equal to itself:
NaN === NaN        // false (!)
Number.isNaN(NaN)  // true  ← correct way to check for NaN

// Checking for NaN (common gotcha):
const result = "hello" * 2;   // NaN
if (result !== result) { }    // Old trick — NaN !== NaN
Number.isNaN(result);         // Modern, correct approach ✅
isNaN("hello");               // true — but isNaN() coerces! Avoid
Number.isNaN("hello");        // false — doesn't coerce ✅
```

**Floating-point precision gotcha:**
```javascript
// Classic IEEE 754 floating-point issue:
0.1 + 0.2             // 0.30000000000000004 (NOT 0.3!)
0.1 + 0.2 === 0.3     // false (!)

// Fix: Use toFixed() for display, or work in integers for money:
(0.1 + 0.2).toFixed(2)    // "0.30" (string)
Math.round((0.1 + 0.2) * 100) / 100  // 0.3 (number)
// For financial apps, work in cents: 10 + 20 = 30 cents, display as $0.30
```

---

#### 2. String

Represents **textual data** — a sequence of characters.

```javascript
typeof "hello"    // "string"
typeof 'world'    // "string"
typeof `template` // "string"

// Three ways to delimit strings (all equivalent for basic use):
const a = "double quotes";
const b = 'single quotes';
const c = `backtick template literal`;
```

**Template literals (ES6):** The preferred modern form — supports multi-line and interpolation:
```javascript
const name = "Vinay";
const age = 25;

// Old way:
const msg1 = "Hello, " + name + "! You are " + age + " years old.";

// Template literal (modern):
const msg2 = `Hello, ${name}! You are ${age} years old.`;
// Any expression works inside ${}:
const msg3 = `2 + 2 = ${2 + 2}`;               // "2 + 2 = 4"
const msg4 = `UPPER: ${name.toUpperCase()}`;    // "UPPER: VINAY"

// Multi-line (no \n needed):
const multiLine = `Line 1
Line 2
Line 3`;
```

**Strings are immutable:**
```javascript
let str = "hello";
str[0] = "H";        // Silently fails! Strings are immutable
console.log(str);    // "hello" — unchanged

// To "modify" a string, you create a NEW string:
str = "H" + str.slice(1);   // "Hello"
```

---

#### 3. Boolean

Represents a **logical true or false** value.

```javascript
typeof true    // "boolean"
typeof false   // "boolean"

let isLoggedIn = true;
let hasPremium = false;
```

Booleans are the result of comparison and logical expressions:
```javascript
10 > 5          // true
"a" === "b"     // false
!true           // false
true && false   // false
true || false   // true
```

---

#### 4. Undefined

Represents the **absence of a value that hasn't been set yet** — the default state of uninitialized things.

```javascript
typeof undefined    // "undefined"

// When does undefined appear?
let x;
console.log(x);              // undefined — declared but not assigned

function greet(name) {
  console.log(name);
}
greet();                     // undefined — parameter not provided

const obj = {};
console.log(obj.missingProp); // undefined — property doesn't exist

function noReturn() {}
console.log(noReturn());     // undefined — function has no return statement
```

---

#### 5. Null

Represents the **intentional absence of a value** — a deliberate "empty" assignment.

```javascript
typeof null    // "object" ← Famous JavaScript bug! (see 8.4)

let user = null;    // Explicitly set to "no user yet"
```

---

#### 6. Symbol (ES6)

Represents a **guaranteed unique identifier** — every Symbol is unique, even if created with the same description.

```javascript
typeof Symbol()    // "symbol"

const id1 = Symbol("id");
const id2 = Symbol("id");
console.log(id1 === id2);    // false — every Symbol is unique!

// Primary use: Unique object property keys (avoid naming collisions)
const MY_KEY = Symbol("myKey");
const obj = {};
obj[MY_KEY] = "secret value";

// Symbol keys don't show up in normal iteration:
console.log(Object.keys(obj));    // [] — Symbol keys are hidden
console.log(obj[MY_KEY]);         // "secret value" — accessible if you have the Symbol
```

**Practical use cases:**
- **Well-known Symbols**: `Symbol.iterator`, `Symbol.hasInstance`, etc. — let you customize built-in JS behaviors
- **Private-ish object keys**: Keys that won't accidentally collide with user-defined or library-defined keys

---

#### 7. BigInt (ES2020)

Represents **integers of arbitrary precision** — for numbers beyond `Number.MAX_SAFE_INTEGER`.

```javascript
typeof 42n      // "bigint" — note the 'n' suffix

const huge = 9007199254740991n + 1n;   // Works perfectly!
const safe = 9007199254740991 + 1;     // 9007199254740992 — may lose precision

// Cannot mix BigInt and Number directly:
42n + 1       // TypeError: Cannot mix BigInt and other types
42n + 1n      // 43n ✅
Number(42n)   // 42 — explicit conversion needed
```

**Use cases**: Cryptography, financial calculations with very large integers, working with 64-bit integer IDs from databases.

---

### The `typeof` Operator — Quick Reference

```javascript
typeof 42              // "number"
typeof 3.14            // "number"
typeof NaN             // "number"  (!)
typeof "hello"         // "string"
typeof true            // "boolean"
typeof undefined       // "undefined"
typeof null            // "object"  ← famous bug
typeof Symbol()        // "symbol"
typeof 42n             // "bigint"
typeof {}              // "object"
typeof []              // "object"  (arrays are objects!)
typeof function(){}    // "function" (special case — functions are objects too)
```

---

## 8.2 Objects & Functions

Everything that is **not a primitive** in JavaScript is an **object**.

### Objects as Keyed Collections

An **object** is a collection of **key-value pairs** (also called **properties**). Keys are strings (or Symbols), values can be anything.

```javascript
// Object literal syntax:
const user = {
  name: "Vinay",          // string key: string value
  age: 25,                // string key: number value
  isActive: true,         // string key: boolean value
  address: {              // string key: nested object value
    city: "Mumbai",
    country: "India"
  },
  greet: function() {     // string key: function value (this is a "method")
    return `Hi, I'm ${this.name}`;
  }
};

// Property access — two syntaxes:
user.name           // "Vinay"    — dot notation (preferred for known keys)
user["name"]        // "Vinay"    — bracket notation (needed for dynamic keys)
user["age"]         // 25
user.address.city   // "Mumbai"   — chained dot access

// Dynamic property access:
const prop = "name";
user[prop]          // "Vinay" — bracket notation with variable key

// Method call:
user.greet()        // "Hi, I'm Vinay"
```

**Objects are mutable and compared by reference:**
```javascript
const a = { x: 1 };
const b = { x: 1 };
a === b    // false — different objects in memory, even with same content!

const c = a;
c === a    // true — c and a point to the SAME object in memory
c.x = 99;
console.log(a.x);  // 99 — modifying c also modifies a! (same reference)
```

**Common Object operations:**
```javascript
const user = { name: "Vinay", age: 25 };

// Add a new property:
user.email = "vinay@example.com";

// Delete a property:
delete user.age;

// Check if a property exists:
"name" in user           // true
"age" in user            // false (deleted)
user.hasOwnProperty("name")  // true

// Get all keys / values / entries:
Object.keys(user)        // ["name", "email"]
Object.values(user)      // ["Vinay", "vinay@example.com"]
Object.entries(user)     // [["name", "Vinay"], ["email", "vinay@example.com"]]

// Shallow copy (spread operator):
const copy = { ...user };    // { name: "Vinay", email: "vinay@example.com" }

// Merge objects:
const merged = { ...user, role: "admin" };
```

---

### First-Class Citizen Functions

In JavaScript, **functions are objects** — they are **first-class citizens** of the language. This means functions can be:

```javascript
// 1. Assigned to variables:
const greet = function(name) {
  return `Hello, ${name}!`;
};

// 2. Passed as arguments to other functions:
function applyToFive(fn) {
  return fn(5);
}
applyToFive(x => x * 2);    // 10
applyToFive(Math.sqrt);      // 2.23...

// 3. Returned from other functions (higher-order functions):
function makeMultiplier(factor) {
  return function(number) {   // Returns a new function!
    return number * factor;
  };
}
const double = makeMultiplier(2);
const triple = makeMultiplier(3);
double(5);   // 10
triple(5);   // 15

// 4. Stored in arrays:
const operations = [Math.sqrt, Math.abs, Math.ceil];
operations[0](16);   // 4

// 5. Stored as object properties (methods):
const calc = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b
};
calc.add(10, 5);    // 15

// 6. Have properties themselves (functions are objects!):
function myFunc() {}
myFunc.customProperty = "I'm a property on a function";
myFunc.callCount = 0;
console.log(myFunc.customProperty);  // "I'm a property on a function"
console.log(myFunc.name);            // "myFunc" — built-in property
console.log(myFunc.length);          // 0 — number of declared parameters
```

**Why First-Class Functions Matter:**

This capability enables powerful programming patterns used constantly in JavaScript:
- **Callbacks**: Pass a function to be called when something happens (event listeners, `setTimeout`, `array.forEach`)
- **Higher-Order Functions**: Functions that operate on other functions (`map`, `filter`, `reduce`)
- **Closures**: Functions that "remember" the scope where they were created
- **Functional Programming**: A programming paradigm built around pure functions and function composition

```javascript
// Real-world first-class function usage:
const numbers = [1, 2, 3, 4, 5];

// Array methods take functions as arguments:
const doubled = numbers.map(n => n * 2);          // [2, 4, 6, 8, 10]
const evens = numbers.filter(n => n % 2 === 0);  // [2, 4]
const sum = numbers.reduce((acc, n) => acc + n, 0); // 15

// Event listener — pass a function to be called on click:
document.getElementById("btn").addEventListener("click", function(event) {
  console.log("Button clicked!", event.target);
});
```

---

## 8.3 Strings & Unicode

### UTF-16 Encoding

JavaScript strings are internally represented as sequences of **UTF-16 code units**. This has important practical implications.

#### The Unicode Scale
- **Unicode** is a universal character standard assigning a unique **code point** to every character
- Code points are written as `U+XXXX` (e.g., `U+0041` = 'A', `U+1F600` = '😀')
- Unicode has over 1.1 million code points, organized into **planes**:
  - **Basic Multilingual Plane (BMP)**: Code points U+0000 to U+FFFF (most common characters)
  - **Supplementary Planes**: U+10000 and above (emoji, rare scripts, ancient characters)

#### UTF-16 and Surrogate Pairs

- **UTF-16** uses **16-bit code units** to represent characters
- BMP characters (U+0000–U+FFFF) fit in **one 16-bit code unit** (1 JS "character")
- Supplementary plane characters (U+10000+) require **two 16-bit code units** — a **surrogate pair**

```javascript
// BMP character — 1 code unit, .length = 1 as expected:
const a = "A";          // U+0041
a.length                // 1 ✅

// Supplementary character (emoji) — 2 code units, .length = 2 unexpectedly:
const emoji = "😀";     // U+1F600 — requires a surrogate pair
emoji.length            // 2 (!) — NOT 1 as a human would expect

// Chinese/Japanese/Korean (CJK) — BMP, so length = 1 per character:
const cjk = "日";       // U+65E5 — BMP character
cjk.length              // 1 ✅

// Emoji string with multiple emoji:
const faces = "😀😂🎉";
faces.length            // 6 (!) — each emoji = 2 code units
```

#### Practical `.length` Edge Cases

```javascript
const text = "Hello, 世界! 🌍";

// .length counts UTF-16 code units, not "characters" as humans see them:
text.length    // 13 (not 11 as you might expect — 🌍 counts as 2)

// Correct character count using spread (ES6):
[...text].length    // 11 ✅ — spread correctly handles surrogate pairs

// Iterating correctly over characters including emoji:
for (const char of text) {
  console.log(char);   // ✅ for...of handles surrogate pairs correctly
}

// Wrong way to iterate — breaks surrogate pairs:
for (let i = 0; i < text.length; i++) {
  console.log(text[i]);  // ❌ May print half of a surrogate pair (gibberish)
}
```

#### ES6 Unicode Escape Syntax

```javascript
// Old escape syntax — only works for BMP (4 hex digits):
"\u0041"    // "A"
"\u4e16"    // "世"

// ES6 syntax — works for ANY code point (any number of hex digits):
"\u{1F600}"  // "😀" — supplementary plane, works with curly braces
"\u{41}"     // "A" — also works for BMP
```

### String Methods — Essential Reference

```javascript
const str = "Hello, World!";

// Information:
str.length              // 13
str.charAt(0)           // "H"
str[0]                  // "H" — bracket access (same result)
str.charCodeAt(0)       // 72 — UTF-16 code unit
str.codePointAt(0)      // 72 — Unicode code point (better for emoji)

// Searching:
str.indexOf("World")     // 7
str.lastIndexOf("l")     // 10
str.includes("Hello")    // true
str.startsWith("Hello")  // true
str.endsWith("!")        // true

// Slicing (non-mutating — returns new string):
str.slice(7, 12)         // "World"
str.slice(-6)            // "orld!" (negative = from end)
str.substring(7, 12)     // "World" (similar to slice, no negatives)

// Transformation (non-mutating):
str.toUpperCase()        // "HELLO, WORLD!"
str.toLowerCase()        // "hello, world!"
str.trim()               // Removes whitespace from both ends
str.trimStart()          // Removes leading whitespace
str.trimEnd()            // Removes trailing whitespace
str.replace("World", "JS")   // "Hello, JS!"
str.replaceAll("l", "L")     // "HeLLo, WorLd!"

// Splitting / Joining:
"a,b,c".split(",")       // ["a", "b", "c"]
["a", "b", "c"].join("-")  // "a-b-c"

// Padding:
"5".padStart(3, "0")     // "005" — useful for formatting
"5".padEnd(3, "0")       // "500"

// Repeating:
"ha".repeat(3)           // "hahaha"
```

---

## 8.4 Non-Values: `undefined` vs. `null`

These two values are among the most confusing aspects of JavaScript, partly because they seem to mean the same thing — but they represent subtly different concepts.

### `undefined` — System-Assigned Absence

`undefined` is the value JavaScript **automatically assigns** when something has no explicit value. It represents an **unintentional or uninitialized absence** — the system is telling you "no value was provided here."

```javascript
// JavaScript creates undefined automatically in these situations:

// 1. Declared variable, not yet assigned:
let name;
console.log(name);         // undefined

// 2. Missing function argument:
function greet(name, greeting) {
  console.log(greeting);
}
greet("Vinay");            // undefined — greeting not provided

// 3. Function with no return statement:
function doNothing() {}
console.log(doNothing());  // undefined

// 4. Accessing a non-existent object property:
const obj = { a: 1 };
console.log(obj.b);        // undefined

// 5. Array element beyond bounds:
const arr = [1, 2, 3];
console.log(arr[10]);      // undefined

// typeof undefined is safe to use even on undeclared variables:
typeof undeclaredVariable  // "undefined" — no ReferenceError!
```

> **The key signal**: `undefined` means "the system doesn't have a value here yet." It's usually **not your fault** — it's the language telling you something is missing.

---

### `null` — Developer-Assigned Absence

`null` is a value you **explicitly assign** to represent "no value" or "empty." It represents an **intentional absence** — the developer is saying "there is no value here, and I mean it."

```javascript
// null is ALWAYS assigned deliberately by a developer:

let currentUser = null;          // No user logged in yet
let selectedItem = null;         // Nothing selected
let pendingRequest = null;       // No pending request

// Typical usage — setting to null before/after use:
let connection = null;           // No connection yet

async function connect() {
  connection = await openConnection();    // Now has a value
}

async function disconnect() {
  await connection.close();
  connection = null;             // Explicitly cleared — "no connection" state
}

// Checking for null vs undefined explicitly:
if (connection !== null) {
  // connection was deliberately set to something
}
```

> **The key signal**: `null` means "I, the developer, have intentionally left this empty." It's always an **explicit programmer decision**.

---

### The Famous `typeof null` Bug

```javascript
typeof null    // "object" ← This is a bug! null is NOT an object.
```

**Why does this exist?** It's a **bug from 1995** in the original JavaScript implementation — values were stored with a type tag, and `null` (represented as a null pointer `0x00`) was accidentally given the "object" type tag. It was caught quickly but could never be fixed without breaking millions of existing websites.

> **TC39 tried to fix it in ES6** — it was proposed to make `typeof null === "null"`, but the proposal was rejected because it would break too much existing code that checks `typeof x === "object"` and expects `null` to be included.

---

### Comparison Behaviors

```javascript
// Strict equality (===) — never coerces:
null === null        // true
undefined === undefined  // true
null === undefined   // false ← They are NOT the same type!

// Loose equality (==) — null and undefined are equal to each other ONLY:
null == undefined    // true  ← Special case!
null == 0            // false
null == ""           // false
null == false        // false
undefined == false   // false
undefined == 0       // false

// This loose equality special case is actually USEFUL:
function process(value) {
  if (value == null) {  // Catches BOTH null AND undefined in one check!
    console.log("No value provided");
    return;
  }
  // Process value...
}
process(null);       // "No value provided"
process(undefined);  // "No value provided"
process(0);          // Not caught — 0 is a valid value
```

### Nullish Coalescing `??` and Optional Chaining `?.` (ES2020)

Modern JavaScript added operators specifically to handle `null`/`undefined` elegantly:

#### Nullish Coalescing `??`
```javascript
// ?? returns the right side ONLY if the left side is null or undefined
// (unlike || which also triggers on 0, "", false)

const name = null ?? "Anonymous";       // "Anonymous"
const age = undefined ?? 0;             // 0
const count = 0 ?? 42;                  // 0  ← 0 is NOT null/undefined!
const label = "" ?? "Default";          // "" ← empty string is NOT null/undefined!

// Compare with || (which replaces falsy values — 0, "", false too):
const count2 = 0 || 42;    // 42 ← WRONG if 0 is a valid value!
const count3 = 0 ?? 42;    // 0  ← CORRECT ✅
```

#### Optional Chaining `?.`
```javascript
// Access deeply nested properties safely — returns undefined instead of throwing
const user = null;

// Without ?. — crashes if user is null:
user.address.city     // TypeError: Cannot read properties of null

// With ?. — safely returns undefined:
user?.address?.city   // undefined ← no crash!

// Works with method calls too:
user?.greet()         // undefined ← no crash even if greet doesn't exist

// Works with bracket notation:
user?.["address"]     // undefined

// Combine with ?? for defaults:
const city = user?.address?.city ?? "Unknown City";
// If user is null, city = "Unknown City"
```

### Stylistic Best Practices Summary

| Situation | Use |
|---|---|
| Variable not yet assigned | `undefined` (let it be assigned by the system) |
| Intentionally clearing a variable | `null` |
| Default value when `null` OR `undefined` | `??` (nullish coalescing) |
| Safe deep property access | `?.` (optional chaining) |
| Checking for "no value" (both null + undefined) | `value == null` (loose equality, valid pattern) |
| Checking for specifically null | `value === null` |
| Checking for specifically undefined | `value === undefined` or `typeof value === "undefined"` |
| Avoid using | `typeof null === "object"` to check for null (it's a bug!) |

---

## Summary of Topic 8

```
DATA TYPES, STRINGS & NON-VALUES

8.1 PRIMITIVE TYPES (7 total — immutable, by value)
  ├── Number: Single type for int + float (IEEE 754 64-bit)
  │     ├── Special: Infinity, -Infinity, NaN
  │     ├── Safe range: ±(2⁵³ - 1) → use BigInt beyond that
  │     └── Gotcha: 0.1 + 0.2 ≠ 0.3 (floating point precision)
  ├── String: UTF-16 sequence, immutable, 3 delimiters (", ', `)
  │     └── Template literals: `${expr}` — preferred modern form
  ├── Boolean: true / false
  ├── Undefined: System-assigned absence (uninitialized/missing)
  ├── Null: Developer-assigned absence (intentional empty)
  ├── Symbol (ES6): Guaranteed unique identifier
  │     └── Use: unique object keys, well-known Symbols
  └── BigInt (ES2020): Arbitrary-precision integers (suffix n: 42n)

8.2 OBJECTS & FUNCTIONS
  ├── Objects: Keyed collections of properties (key-value pairs)
  │     ├── Compared by REFERENCE (not value like primitives)
  │     ├── Mutable — properties can be added/changed/deleted
  │     └── Operations: keys(), values(), entries(), spread {...obj}
  └── Functions: First-class citizens (objects with a () call ability)
        ├── Assignable to variables
        ├── Passable as arguments (callbacks)
        ├── Returnable from other functions (higher-order functions)
        ├── Storable in arrays and object properties (methods)
        └── Can have properties attached directly

8.3 STRINGS & UNICODE
  ├── Internally UTF-16 code units (16-bit)
  ├── BMP chars (U+0000–U+FFFF): 1 code unit, .length = 1 ✅
  ├── Supplementary chars (emoji, etc.): 2 code units (surrogate pair)
  │     └── Gotcha: "😀".length === 2, not 1!
  ├── Correct iteration: for...of or spread [...str] handles surrogates
  └── ES6 Unicode: "\u{1F600}" (curly brace syntax for any code point)

8.4 NULL vs. UNDEFINED
  ├── undefined: System-assigned (uninitialized var, missing arg, no return)
  ├── null: Developer-assigned (intentional empty — you set this!)
  ├── typeof null === "object" ← famous historical bug (never fixed)
  ├── null == undefined → true (loose equality special case — useful!)
  ├── null === undefined → false (strict equality)
  └── Modern operators:
        ├── ?? (nullish coalescing): right side if left is null/undefined
        └── ?. (optional chaining): safe deep access, returns undefined
```
