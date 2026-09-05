# 2. Modularity & Object-Oriented JavaScript

---

## 2.1 Modules Overview

### The Problem Modules Solve
Before modules, all JavaScript lived in the **global scope**. Every script tag dumped its variables and functions into one shared namespace. This caused:
- **Name collisions** — two libraries both defining `utils` would overwrite each other
- **Load order dependencies** — script B could silently break if script A hadn't loaded first
- **No encapsulation** — all internals were exposed; nothing was truly "private"

> A **module** is a self-contained unit of code that **encapsulates** its own scope, exposes a deliberate public interface, and explicitly declares what it depends on.

---

### The Three Pillars of a Module System

#### ① Encapsulation
Group related functions, objects, and values into a single logical unit. Everything inside the module is **private by default** — only what you explicitly export is accessible from outside.

#### ② `export` — Defining the Public Interface
Mark what you want to make available to the outside world:

```javascript
// math.js
const PI = 3.14159;

export function add(a, b) { return a + b; }
export function multiply(a, b) { return a * b; }
// PI is NOT exported — it's private to this module
```

#### ③ `import` — Consuming Dependencies
Declare exactly what you need from other modules:

```javascript
// app.js
import { add, multiply } from './math.js';

console.log(add(2, 3));       // 5
console.log(multiply(4, 5));  // 20
```

> This makes dependencies **explicit and auditable** — you can see exactly what each file depends on, without hunting through global state.

---

## 2.2 Evolution and Implementation of Modules

JavaScript did not always have a built-in module system. The ecosystem evolved through several stages:

---

### Stage 1 — `<script>` Tags (No Modules)
The original approach: include scripts directly in HTML.

```html
<script src="utils.js"></script>
<script src="app.js"></script>
```

- Everything lands in the **global `window` object**
- Scripts must be loaded in the **correct order** manually
- No isolation — any script can overwrite any other's variables
- **Still used today** for simple pages, but does not scale

---

### Stage 2 — CommonJS (`require` / `module.exports`)
Created for **Node.js** (server-side). CommonJS introduced the first widely-adopted module system.

```javascript
// math.js  (exporting)
function add(a, b) { return a + b; }
module.exports = { add };

// app.js  (importing)
const { add } = require('./math.js');
console.log(add(2, 3));  // 5
```

| Property | CommonJS |
|----------|----------|
| **Environment** | Node.js (server-side) |
| **Loading** | **Synchronous** — blocks until file is loaded |
| **Syntax** | `require()` / `module.exports` |
| **Works in browser?** | ❌ Not natively (needs bundling) |

> **Why synchronous is fine on the server**: On the server, files are loaded from local disk (fast). In the browser, files are fetched over a network — synchronous loading would freeze the page.

---

### Stage 3 — AMD (Asynchronous Module Definition)
Designed specifically for the **browser**. Loads modules asynchronously so the page doesn't freeze.

```javascript
// AMD syntax (RequireJS library)
define(['dependency1', 'dependency2'], function(dep1, dep2) {
  return {
    myFunction: function() { /* ... */ }
  };
});
```

| Property | AMD |
|----------|-----|
| **Environment** | Browser (client-side) |
| **Loading** | **Asynchronous** |
| **Syntax** | `define()` / `require()` (RequireJS) |
| **Downside** | Verbose, non-standard syntax; required external library |

---

### Stage 4 — ES6 Modules (`import` / `export`) ✅ Modern Standard
ES2015 (ES6) introduced a **native, standardized** module system into the JavaScript language itself. This is what you should use today.

```javascript
// Named exports
export const PI = 3.14;
export function square(x) { return x * x; }

// Default export (one per module)
export default class Calculator { /* ... */ }
```

```javascript
// Named imports
import { PI, square } from './math.js';

// Default import
import Calculator from './math.js';

// Import everything as a namespace
import * as MathUtils from './math.js';
MathUtils.square(4);  // 16
```

| Property | ES6 Modules |
|----------|-------------|
| **Environment** | Browser + Node.js |
| **Loading** | **Asynchronous** (browser) |
| **Syntax** | `import` / `export` (native keywords) |
| **Static analysis** | ✅ Imports resolved at parse time (enables tree-shaking) |
| **Use in HTML** | `<script type="module" src="app.js">` |

---

### Comparison Table

| Feature | `<script>` | CommonJS | AMD | ES6 Modules |
|---------|-----------|----------|-----|-------------|
| Environment | Browser | Node.js | Browser | Both |
| Loading | Synchronous | Synchronous | Asynchronous | Asynchronous |
| Scope isolation | ❌ Global | ✅ | ✅ | ✅ |
| Static analysis | ❌ | ❌ | ❌ | ✅ |
| Native to JS | ❌ | ❌ | ❌ | ✅ |

---

## 2.3 Node.js & npm Ecosystem

### Node.js — JavaScript Beyond the Browser
> **Node.js** is a JavaScript runtime built on Chrome's V8 engine that lets you run JavaScript **outside of a browser** — on the command line, on servers, or in build tools.

**Why it matters for frontend developers:**
- Run JavaScript build tools locally (Webpack, Vite, Babel)
- Write backend APIs in the same language as the frontend
- Use npm packages in your development workflow
- Run test suites from the terminal

```bash
# Run a JS file from terminal
node app.js
```

---

### npm — Node Package Manager
> **npm** is the default package manager for Node.js. It lets you install, manage, and publish reusable JavaScript libraries (packages).

```bash
npm install lodash          # install a package
npm install --save-dev jest # install as dev dependency
npm uninstall lodash        # remove a package
npm run build               # run a script defined in package.json
```

**`package.json`** — every Node.js project has this file. It records:
- Project metadata (name, version, description)
- `dependencies` — packages needed at runtime
- `devDependencies` — packages needed only during development (testing, building)
- `scripts` — shorthand commands (`npm run dev`, `npm test`)

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "jest"
  },
  "dependencies": {
    "react": "^18.0.0"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "jest": "^29.0.0"
  }
}
```

---

### Module Bundlers
Browsers cannot natively import thousands of small files efficiently. **Module bundlers** solve this by combining all your modules into one (or a few) optimized output files.

| Bundler | Notes |
|---------|-------|
| **Webpack** | Most established, highly configurable |
| **Rollup** | Excellent for libraries; pioneered tree-shaking |
| **Vite** | Modern, extremely fast (uses native ES modules in dev) |
| **Parcel** | Zero-config bundler |

> **Tree-shaking**: A bundler optimization that removes unused exports from the final bundle. Only possible with ES6 Modules (because imports are statically analyzable).

---

## 2.4 Objects & Function Context

### Everything is an Object (Almost)
In JavaScript, nearly everything is an object or behaves like one:
- Arrays are objects (`typeof [] === 'object'`)
- Functions are objects (`typeof function(){} === 'function'`, but they have object properties)
- Even primitives like strings get object-like behavior temporarily via **boxing**

---

### Object Literals
The simplest way to create an object — a comma-separated list of key-value pairs in `{}`:

```javascript
const person = {
  name: "Alice",
  age: 30,
  greet: function() {
    console.log("Hello, I'm " + this.name);
  },
  // ES6 shorthand method syntax:
  farewell() {
    console.log("Goodbye from " + this.name);
  }
};

person.name;       // "Alice"
person["age"];     // 30  ← bracket notation (useful for dynamic keys)
person.greet();    // "Hello, I'm Alice"
```

---

### The `this` Keyword & Execution Context
> **`this`** refers to the **object that is currently executing the function** — the "execution context".

`this` is **not** determined by where a function is defined, but by **how it is called**:

```javascript
const obj = {
  name: "Alice",
  greet() {
    console.log(this.name);  // "this" = obj → "Alice"
  }
};
obj.greet();

// But if you detach the function...
const fn = obj.greet;
fn();   // "this" = undefined (strict mode) or window (non-strict) → NOT "Alice"
```

This is one of the most common sources of bugs in JavaScript.

**`this` in arrow functions:**
Arrow functions do **not** have their own `this` — they inherit `this` from the surrounding lexical scope:

```javascript
const obj = {
  name: "Alice",
  greet() {
    const inner = () => {
      console.log(this.name);  // inherits "this" from greet() → "Alice" ✅
    };
    inner();
  }
};
obj.greet();
```

---

### Controlling `this`: `call()`, `apply()`, `bind()`
These three methods let you **explicitly set** what `this` refers to when calling a function.

#### `call(thisArg, arg1, arg2, ...)` — Call Immediately with Arguments
```javascript
function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

const alice = { name: "Alice" };
const bob   = { name: "Bob" };

greet.call(alice, "Hello", "!");   // "Hello, Alice!"
greet.call(bob,   "Hi",    ".");   // "Hi, Bob."
```

#### `apply(thisArg, [argsArray])` — Call Immediately with Arguments as Array
```javascript
greet.apply(alice, ["Hello", "!"]);  // "Hello, Alice!"
// Same as call(), but arguments are passed as an array
```

> **Memory tip**: `call` = **C**omma-separated args. `apply` = **A**rray of args.

#### `bind(thisArg, ...args)` — Returns a New Function (Doesn't Call Immediately)
```javascript
const greetAlice = greet.bind(alice, "Hello");  // pre-set this AND first arg
greetAlice("!");   // "Hello, Alice!"
greetAlice("?");   // "Hello, Alice?"
```

`bind` is commonly used to preserve `this` when passing methods as callbacks:
```javascript
class Timer {
  constructor() { this.seconds = 0; }
  
  start() {
    // Without bind, `this` inside the callback would be wrong
    setInterval(this.tick.bind(this), 1000);
  }
  
  tick() { console.log(++this.seconds); }
}
```

---

### Object Utility Functions
Quick recap of the three static helpers:

```javascript
const obj = { a: 1, b: 2, c: 3 };

Object.keys(obj);     // ["a", "b", "c"]    — array of property names
Object.values(obj);   // [1, 2, 3]           — array of property values
Object.entries(obj);  // [["a",1],["b",2],["c",3]]  — array of [key, value] pairs
```

---

## 2.5 Prototype-based Inheritance

### JavaScript's Inheritance Model
JavaScript uses **prototype-based inheritance** — objects inherit directly from other objects, not from classes (even though ES6 classes look class-based, they're syntactic sugar over prototypes).

Every JavaScript object has an internal `[[Prototype]]` link (accessible via `__proto__` or `Object.getPrototypeOf()`). When you access a property that doesn't exist on the object itself, JavaScript **walks up the prototype chain** looking for it.

---

### The Prototype Chain

```javascript
const animal = {
  breathe() { console.log("breathing..."); }
};

const dog = {
  bark() { console.log("Woof!"); }
};

// Set animal as the prototype of dog
Object.setPrototypeOf(dog, animal);

dog.bark();     // "Woof!"    ← found on dog itself
dog.breathe();  // "breathing..." ← NOT on dog, found on animal via prototype
dog.toString(); // "[object Object]" ← found further up on Object.prototype
```

The chain: `dog` → `animal` → `Object.prototype` → `null`

---

### Property Lookup (Delegation)
When you access `obj.prop`:
1. Check `obj` itself for `prop`
2. If not found, check `obj.__proto__` (its prototype)
3. If not found, check `obj.__proto__.__proto__` (prototype's prototype)
4. Continue until `null` is reached → return `undefined`

This is called **property delegation** — child objects delegate unknown lookups to their prototype.

---

### Single Inheritance
JavaScript's prototype chain is **linear** — each object has exactly one prototype. There is no native multiple inheritance (one object cannot directly inherit from two separate prototypes).

```
MyClass → ParentClass → GrandparentClass → Object.prototype → null
```

> (Multiple inheritance can be partially simulated with **mixins** — covered in 2.6)

---

## 2.6 ES6 Classes

### Classes as Syntactic Sugar
ES6 classes do **not** introduce a new inheritance model. They are syntactic sugar over the existing prototype-based system — more readable and familiar to developers coming from Java/Python/C++.

```javascript
// Old way (constructor function + prototype)
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function() {
  console.log(this.name + " makes a sound.");
};

// New way (ES6 class — equivalent result)
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(`${this.name} makes a sound.`);
  }
}

const a = new Animal("Cat");
a.speak();  // "Cat makes a sound."
```

Under the hood, `speak` is still placed on `Animal.prototype`.

---

### Class Anatomy

```javascript
class Vehicle {
  // Static property (belongs to the class, not instances)
  static count = 0;

  // Constructor — called with `new Vehicle(...)`
  constructor(make, model) {
    this.make = make;    // instance property
    this.model = model;
    Vehicle.count++;
  }

  // Instance method (on prototype)
  describe() {
    return `${this.make} ${this.model}`;
  }

  // Static method (called on the class, not instances)
  static getCount() {
    return Vehicle.count;
  }
}

const car = new Vehicle("Toyota", "Corolla");
car.describe();         // "Toyota Corolla"
Vehicle.getCount();     // 1
```

---

### Subclassing with `extends`
Create a child class that inherits from a parent:

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(`${this.name} makes a sound.`);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);         // ← MANDATORY: must call super() before using `this`
    this.breed = breed;
  }

  // Override parent method
  speak() {
    console.log(`${this.name} barks!`);
  }

  // New method only on Dog
  fetch() {
    console.log(`${this.name} fetches the ball.`);
  }
}

const d = new Dog("Rex", "Labrador");
d.speak();   // "Rex barks!"     ← overridden
d.fetch();   // "Rex fetches the ball."
```

> **`super()`** — In a subclass constructor, `super()` calls the parent class constructor. It **must** be called before you can access `this`. Forgetting it throws a `ReferenceError`.

---

### `super` for Method Calls
You can also use `super.methodName()` to call the parent's version of an overridden method:

```javascript
class Dog extends Animal {
  speak() {
    super.speak();   // calls Animal.speak() → "Rex makes a sound."
    console.log(`${this.name} barks too!`);
  }
}
```

---

### Multiple Inheritance & Mixins
JavaScript classes support only **single inheritance** (`extends` one class). To reuse behavior from multiple sources, use **mixins** — plain functions that add methods to a class's prototype:

```javascript
// Mixin: a function that returns a class with extra methods
const Serializable = (Base) => class extends Base {
  serialize() {
    return JSON.stringify(this);
  }
};

const Validatable = (Base) => class extends Base {
  validate() {
    return Object.keys(this).every(k => this[k] !== null);
  }
};

class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

// Apply both mixins
class EnhancedUser extends Serializable(Validatable(User)) {}

const u = new EnhancedUser("Alice", "alice@example.com");
u.serialize();  // '{"name":"Alice","email":"alice@example.com"}'
u.validate();   // true
```

> Mixins are a common pattern for achieving mixin-style composition without true multiple inheritance.

---

## Summary

| Concept | Key Idea | Key Syntax / Tool |
|---------|----------|-------------------|
| **Modules** | Encapsulate code, explicit imports/exports | `import` / `export` |
| **Script tag** | Global scope, no isolation | `<script src="...">` |
| **CommonJS** | Sync loading, Node.js standard | `require()` / `module.exports` |
| **AMD** | Async loading for browsers | `define()` (RequireJS) |
| **ES6 Modules** | Native, static, async, tree-shakeable | `import` / `export` |
| **Node.js** | JS runtime outside browser | `node app.js` |
| **npm** | Package manager + scripts | `npm install`, `package.json` |
| **Bundlers** | Combine modules for browser delivery | Webpack, Vite, Rollup |
| **`this`** | Execution context of a function | Depends on call site |
| **`call/apply/bind`** | Explicitly set `this` | `.call()`, `.apply()`, `.bind()` |
| **Prototypes** | Delegation-based inheritance chain | `[[Prototype]]`, `__proto__` |
| **ES6 Classes** | Syntactic sugar over prototypes | `class`, `extends`, `super()` |
| **Mixins** | Simulate multiple inheritance | HOF returning a class |
