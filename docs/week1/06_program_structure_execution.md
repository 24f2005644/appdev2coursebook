# Topic 6: JavaScript Program Structure & Execution

> **MAD-II Week 1 | Detailed Notes**

---

## Overview

Before writing a single line of JavaScript, it's important to understand *where* it runs and *how* it's structured. Unlike Python (which runs as a script file), Java (which requires a class with `main()`), or C (which requires a `main()` function), JavaScript is remarkably flexible — and intentionally loose — about both its execution environment and its structural requirements.

---

## 6.1 Execution Contexts

JavaScript can execute in two fundamentally different contexts. The environment determines what APIs are available, how code is loaded, and what the global object is.

---

### Context 1: Frontend — Inside HTML Documents (Browser)

#### The `<script>` Tag

The most common way to run JavaScript in a browser is via the `<script>` tag embedded in an HTML document.

##### Inline Script
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <title>My Page</title>
</head>
<body>
  <h1 id="greeting">Hello</h1>

  <script>
    // JavaScript runs here, after the HTML above has been parsed
    const heading = document.getElementById("greeting");
    heading.textContent = "Hello from JavaScript!";
  </script>
</body>
```

##### External Script File (Preferred)
```html
<!-- Link to a separate .js file -->
<script src="app.js"></script>

<!-- With defer: downloads in parallel, executes after DOM is fully parsed -->
<script src="app.js" defer></script>

<!-- With async: downloads in parallel, executes as soon as it's downloaded -->
<script src="app.js" async></script>
```

#### Script Loading Strategies — Critical Detail

The **placement and attributes** of `<script>` tags dramatically affect both performance and correctness:

```
HTML Parsing:    ██████░░░░░░░░░░░██████████████████████████
                        ↑
                  Hits <script>
                  without defer/async

No attribute (in <head>):
  HTML Parse: ████│                              │████████
                  │ Script download + execute   │
                  └─────────────────────────────┘
  ❌ Blocks HTML parsing — page appears blank until script loads

No attribute (at end of <body>) — Legacy best practice:
  HTML Parse: ████████████████████████████████│
                                              │ Script download + execute
  ✅ Page content visible first, but script loads last

defer (anywhere):
  HTML Parse: ██████████████████████████████████│
  Script DL:  ████████████████│                 │ Execute
                               (Downloaded in   │ (After DOM ready)
                                parallel)
  ✅ Best for most scripts — parallel download, executes after DOM is ready

async (anywhere):
  HTML Parse: ████│        │███████████████████
  Script DL:  ████│███████ Execute (immediately when downloaded)
  ✅ Good for independent scripts (analytics, ads) — order not guaranteed
```

| Attribute | Download | Execute When | Blocks HTML? | Order Preserved? |
|---|---|---|---|---|
| None (in `<head>`) | Immediate | Immediately | ✅ Yes | ✅ Yes |
| None (end of `<body>`) | After HTML | Immediately | ❌ No | ✅ Yes |
| `defer` | Parallel | After DOM ready | ❌ No | ✅ Yes |
| `async` | Parallel | When downloaded | ❌ No | ❌ No |

#### ES6 Modules in the Browser

Modern JavaScript uses native **ES6 modules** with `type="module"`:

```html
<script type="module" src="main.js"></script>
```

```javascript
// main.js
import { greet } from './utils.js';
greet("Vinay");

// utils.js
export function greet(name) {
  console.log(`Hello, ${name}!`);
}
```

Key differences from regular scripts:
- **Automatically deferred** — behaves like `defer` by default
- **Automatically in strict mode** — no `"use strict"` needed
- **Own scope** — variables declared in a module don't leak to `window`
- **Same-origin policy** — modules must be served over HTTP/HTTPS (not `file://`)

#### What the Browser Context Provides

When JS runs in the browser, it has access to browser-specific global APIs that don't exist in Node.js:

| API | Purpose |
|---|---|
| `document` | The DOM — access and manipulate the HTML tree |
| `window` | The global object — also hosts `alert()`, `setTimeout()`, `fetch()` |
| `navigator` | Browser/device information (user agent, geolocation, etc.) |
| `location` | Current URL, redirect to other pages |
| `localStorage` / `sessionStorage` | Persistent key-value storage in the browser |
| `XMLHttpRequest` / `fetch` | Make HTTP requests |
| `alert()` / `confirm()` / `prompt()` | Simple blocking dialog boxes |
| `addEventListener` | Listen for user events (click, keypress, etc.) |

---

### Context 2: Backend / Headless — Command-Line via Node.js

**Node.js** allows JavaScript to run entirely outside the browser, on a server or local machine via the command line.

#### Running JS with Node.js

```bash
# Run a JavaScript file
node app.js

# Open the Node.js REPL (interactive shell, like Python's >>> prompt)
node

# Run a one-liner
node -e "console.log('Hello from the command line!')"
```

#### The Node.js REPL

The Node.js **REPL** (Read-Evaluate-Print Loop) is an interactive JS shell — equivalent to Python's `>>>` prompt:

```
$ node
Welcome to Node.js v20.0.0.
Type ".help" for more information.
> 2 + 2
4
> const name = "Vinay"
undefined
> `Hello, ${name}!`
'Hello, Vinay!'
> [1, 2, 3].map(x => x * 2)
[ 2, 4, 6 ]
> .exit
$
```

Perfect for quick experiments, testing expressions, and debugging.

#### What Node.js Provides (vs. Browser)

Node.js replaces browser APIs with server/OS-oriented APIs:

| Browser API | Node.js Equivalent | Purpose |
|---|---|---|
| `document`, `window` | ❌ Not available | No DOM in Node.js |
| `fetch` | `https` module / `node-fetch` | HTTP requests |
| `localStorage` | `fs` module (file system) | Persistent storage |
| — | `fs` (File System) | Read/write files |
| — | `path` | Work with file paths |
| — | `os` | Operating system info |
| — | `http` / `https` | Create HTTP servers |
| — | `child_process` | Run subprocesses |
| — | `cluster` | Multi-process scaling |

```javascript
// Node.js: Read a file (impossible in the browser sandbox)
const fs = require('fs');
const content = fs.readFileSync('data.txt', 'utf8');
console.log(content);

// Node.js: Create an HTTP server
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Node.js!');
});
server.listen(3000, () => console.log('Server on port 3000'));
```

#### Other Headless / Online JS Environments

| Tool | Description | Best For |
|---|---|---|
| **Node.js** | Local CLI runtime | Backend servers, scripts, tooling |
| **Deno** | Secure Node.js alternative | Scripts, modern backend |
| **Replit** | Online IDE with JS/Node.js | Quick prototyping, sharing code |
| **Browser DevTools Console** | Built into Chrome/Firefox | Testing snippets, debugging |
| **JS Fiddle / CodePen / StackBlitz** | Online browser-based playgrounds | Frontend experimentation |
| **Bun** | Fast JS runtime (alternative to Node) | Performance-critical tooling |

#### The Global Object: `window` vs. `global` vs. `globalThis`

The global object differs between environments — `globalThis` is the modern universal solution:

| Environment | Global Object | Access |
|---|---|---|
| Browser | `window` | `window.setTimeout`, `window.fetch` |
| Node.js | `global` | `global.setTimeout`, `global.process` |
| Web Workers | `self` | `self.postMessage` |
| **Any (ES2020+)** | `globalThis` | ✅ Works everywhere |

```javascript
// Safe global access — works in browser, Node.js, Web Workers, and Deno:
globalThis.setTimeout(() => console.log("Universal timer"), 1000);
```

---

## 6.2 Program Structure

### No Mandatory Entry Point

This is one of the most striking differences from other languages:

```python
# Python: No mandatory structure, but conventional
def main():
    print("Hello")

if __name__ == "__main__":
    main()
```

```java
// Java: MANDATORY class + main method — won't compile without it
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello");
    }
}
```

```c
// C: MANDATORY main() function — entry point required
int main() {
    printf("Hello\n");
    return 0;
}
```

```javascript
// JavaScript: No required structure at all!
// This is a complete, valid, runnable JavaScript program:
console.log("Hello, World!");

// Or even:
2 + 2   // A valid (if useless) JavaScript program — just an expression statement
```

JavaScript is a **scripting language** — the entire file is executed top-to-bottom as a sequence of statements. There is no mandatory `main()` function, no required class, no entry point declaration.

#### Implications of the Loose Structure

| Implication | Detail |
|---|---|
| **Quick to start** | No boilerplate — write one line and it runs |
| **Global scope by default** | Variables declared at the top level are global (problem in large apps) |
| **Execution order matters** | Code runs top-to-bottom; using a variable before it's declared can error |
| **Modules solve structure** | ES6 `import`/`export` adds structured organization to large codebases |
| **Frameworks impose structure** | React, Vue, Angular add their own structural conventions |

#### Conventional Structure in Real Projects

While nothing is mandatory, real JavaScript projects follow conventions:

```javascript
// 1. Imports (ES6 modules)
import { fetchUser } from './api.js';
import { renderCard } from './components.js';

// 2. Constants / Configuration
const API_URL = 'https://api.example.com';
const MAX_RETRIES = 3;

// 3. Helper function definitions
function formatDate(dateString) {
  return new Date(dateString).toLocaleDateString();
}

// 4. Main application logic
async function init() {
  const user = await fetchUser(42);
  renderCard(user);
}

// 5. Kickoff (equivalent of "main")
init();
```

This structure is convention, not enforced by the language.

---

### Comments

Comments are non-executing annotations in code — ignored by the JavaScript engine. They serve documentation, explanation, and temporary code disabling purposes.

#### Single-Line Comments: `//`

```javascript
// This is a single-line comment — everything after // is ignored
const age = 25; // Inline comment — same line as code

// Common uses:
// 1. Explain WHY (not just what — the code shows what)
// 2. Document parameters or return values (where JSDoc isn't used)
// 3. Temporarily disable a line during debugging
// console.log("debug output");  ← Commented-out code
```

#### Multi-Line Comments: `/* ... */`

```javascript
/*
  This is a multi-line comment.
  It can span as many lines as needed.
  Use for longer explanations or to comment out blocks of code.
*/

/* Can also be used inline */ const x = 10; /* like this */

/*
  Commented-out block for debugging:
  
  const result = expensiveOperation();
  console.log("Intermediate:", result);
  processResult(result);
*/
```

#### JSDoc Comments: `/** ... */`

A special convention for documenting functions — used by IDEs and documentation generators:

```javascript
/**
 * Calculates the total price including tax.
 * @param {number} price - The base price before tax
 * @param {number} taxRate - The tax rate as a decimal (e.g., 0.18 for 18%)
 * @returns {number} The total price including tax, rounded to 2 decimal places
 * @example
 * calculateTotal(100, 0.18); // returns 118.00
 */
function calculateTotal(price, taxRate) {
  return Math.round((price * (1 + taxRate)) * 100) / 100;
}
```

IDEs like VS Code read JSDoc comments and show them in autocomplete tooltips — even for plain JavaScript files (no TypeScript needed).

#### Comment Best Practices

| ✅ Good Comments | ❌ Bad Comments |
|---|---|
| Explain *why* a decision was made | Restate what the code obviously does |
| Document non-obvious side effects | Comment every single line |
| Warn about gotchas or known issues | Leave large blocks of dead code commented out |
| Link to relevant docs, tickets, specs | Leave TODO comments that never get resolved |
| JSDoc for public API functions | Use comments to compensate for unclear variable names |

```javascript
// ❌ Bad — states the obvious:
let count = 0; // set count to zero

// ✅ Good — explains WHY:
let count = 0; // Start at 0, not 1, because the API uses 0-based pagination

// ❌ Bad — compensates for bad naming:
let x = 86400; // seconds in a day

// ✅ Good — name says it all, no comment needed:
const SECONDS_PER_DAY = 86400;
```

---

## Summary of Topic 6

```
JAVASCRIPT PROGRAM STRUCTURE & EXECUTION

6.1 EXECUTION CONTEXTS
  │
  ├── FRONTEND: Browser / <script> tag
  │     ├── Inline: <script>...</script>
  │     ├── External: <script src="app.js"></script>
  │     ├── Loading strategies:
  │     │     ├── No attr in <head> → BLOCKS HTML parsing ❌
  │     │     ├── No attr at end of <body> → Legacy safe ✅
  │     │     ├── defer → Parallel download, execute after DOM ✅ (preferred)
  │     │     └── async → Parallel download, execute immediately on load
  │     ├── type="module" → Auto-defer, auto-strict, own scope
  │     └── Browser APIs: document, window, fetch, localStorage, navigator...
  │
  ├── BACKEND: Node.js (CLI / Server)
  │     ├── Run: node app.js
  │     ├── REPL: node (interactive shell)
  │     ├── Node.js APIs: fs, path, http, os, child_process
  │     └── No DOM, no window — different global object ("global")
  │
  ├── OTHER ENVIRONMENTS: Replit, Browser DevTools, Deno, Bun, CodePen
  │
  └── GLOBAL OBJECT
        ├── Browser: window
        ├── Node.js: global
        ├── Workers: self
        └── Universal (ES2020+): globalThis ✅

6.2 PROGRAM STRUCTURE
  │
  ├── NO MANDATORY ENTRY POINT
  │     ├── No main() required (unlike Java, C)
  │     ├── No class wrapping required
  │     └── Top-to-bottom scripting execution model
  │
  ├── CONVENTIONAL STRUCTURE (not enforced):
  │     imports → constants → helpers → main logic → kickoff call
  │
  └── COMMENTS
        ├── Single-line: // comment text
        ├── Multi-line:  /* comment block */
        └── JSDoc:       /** @param, @returns, @example */
              └── IDE tooltip documentation for functions
```
