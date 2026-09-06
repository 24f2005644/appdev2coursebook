# Topic 7: Identifiers, Statements, Expressions & Grammar



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 7: Identifiers, Statements, Expressions & Grammar**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

> **MAD-II Week 1 | Detailed Notes**

---

## Overview

Every programming language has a **grammar** — a formal set of rules that define what combinations of characters form valid programs. JavaScript's grammar is the foundation that the parser uses to make sense of your code. Understanding the grammar at this level removes the mystery from many confusing behaviors and error messages you'll encounter.

This topic covers the three layers of JavaScript grammar:
1. **Identifiers** — the names used for variables, functions, classes, etc.
2. **Statements** — the instructions that *do* things
3. **Expressions** — the code units that *produce values*

---

## 7.1 Identifiers & Keywords

### What is an Identifier?

An **identifier** is a name used to refer to a variable, function, class, parameter, property, or label in your code.

```javascript
let userName = "Vinay";         // "userName" is an identifier
function calculateTotal() {}    // "calculateTotal" is an identifier
class UserProfile {}            // "UserProfile" is an identifier
const MAX_SIZE = 100;           // "MAX_SIZE" is an identifier
```

### Identifier Rules

JavaScript identifiers must follow these rules:

| Rule | Valid | Invalid |
|---|---|---|
| Must start with a letter, `_`, or `$` | `name`, `_temp`, `$el` | `1name`, `-val`, `@tag` |
| After first char: letters, digits, `_`, `$` | `user1`, `my_var`, `$btn2` | `my-var`, `my var` |
| Case-sensitive | `name` ≠ `Name` ≠ `NAME` | (all three are distinct) |
| No reserved keywords | `let myVar` | `let let`, `let if` |
| Unicode allowed (but avoid) | `let café`, `let 変数` | (technically valid, avoid in practice) |

```javascript
// Valid identifiers:
let name = "Vinay";
let _privateVar = 42;
let $jQueryStyle = true;
let camelCaseIsConvention = "yes";
let SCREAMING_SNAKE_FOR_CONSTANTS = 3.14;

// Invalid identifiers (SyntaxErrors):
let 1stPlace = "gold";    // SyntaxError: starts with a digit
let my-variable = 10;     // SyntaxError: hyphen not allowed
let my variable = 10;     // SyntaxError: space not allowed
```

### Naming Conventions (Not Enforced, But Universal)

| Convention | Style | Used For |
|---|---|---|
| `camelCase` | `getUserName`, `calculateTotal` | Variables, functions, methods |
| `PascalCase` | `UserProfile`, `ShoppingCart` | Classes, constructors, React components |
| `SCREAMING_SNAKE_CASE` | `MAX_RETRIES`, `API_BASE_URL` | Constants with fixed values |
| `_prefixed` | `_internalHelper` | Convention for "private" (not enforced by language) |
| `$prefixed` | `$element`, `$` | jQuery style, or auto-generated identifiers |

---

### Categories of Special Words

JavaScript has several categories of words that are either **reserved**, **literal**, or **restricted** — meaning you cannot use them freely as identifiers.

---

### Category 1: Reserved Keywords

These words are part of JavaScript's **syntax** — the parser uses them to understand the structure of your program. They **cannot** be used as variable names, function names, or class names.

#### Declaration Keywords
```javascript
var     // Legacy variable declaration (function-scoped)
let     // Block-scoped variable declaration (ES6)
const   // Block-scoped constant declaration (ES6)
```

#### Function & Class Keywords
```javascript
function   // Function declaration/expression
class      // Class declaration/expression
new        // Create an instance of a class/constructor
return     // Return a value from a function
this       // Reference to current execution context
super      // Reference to parent class
extends    // Class inheritance
```

#### Control Flow Keywords
```javascript
if        // Conditional execution
else      // Alternative branch of if
switch    // Multi-way branching
case      // Branch label in switch
default   // Default case in switch / default export
break     // Exit a loop or switch
continue  // Skip current loop iteration
for       // Loop construct
while     // Loop construct
do        // do...while loop
```

#### Error Handling Keywords
```javascript
try       // Attempt a block that might throw
catch     // Handle a thrown error
finally   // Always execute (cleanup) after try/catch
throw     // Throw an error/exception
```

#### Module Keywords
```javascript
import    // Import from a module
export    // Export from a module
```

#### Other Structural Keywords
```javascript
delete    // Remove a property from an object
typeof    // Get the type of a value as a string
instanceof // Check if object is instance of class
in        // Check if property exists in object
void      // Evaluate expression, return undefined
with      // (Banned in strict mode — avoid entirely)
yield     // Pause a generator function
await     // Await a Promise inside async function
async     // Mark a function as asynchronous
static    // Define a static class member
get       // Define a property getter
set       // Define a property setter
```

> **Attempting to use a keyword as an identifier:**
> ```javascript
> let return = 5;    // SyntaxError: Unexpected token 'return'
> let if = true;     // SyntaxError: Unexpected token 'if'
> let class = {};    // SyntaxError: Unexpected token 'class'
> ```

---

### Category 2: Literal Values

These are not keywords in the traditional sense — they are **special literal expressions** that represent fixed values in the language. They also cannot be reassigned.

```javascript
true    // Boolean true literal
false   // Boolean false literal
null    // The null literal (intentional absence of value)
```

```javascript
// These are values, not variables — you cannot reassign them:
true = 1;    // SyntaxError (in strict mode) or silently fails (sloppy mode)
null = {};   // SyntaxError

// But you CAN (unfortunately) shadow them in old code — don't do this:
// var undefined = "oops";  // Works in sloppy mode — extremely dangerous
```

---

### Category 3: Disallowed / Reserved Future Words

These fall into a nuanced middle ground — they are **not current keywords** (the parser doesn't use them today), but they are **reserved for future use** or have restrictions in specific contexts.

#### Strict Mode Reserved Words
These are regular identifiers in sloppy mode but **reserved in strict mode**:
```
implements   interface   let   package
private      protected   public   static   yield
```

```javascript
// In sloppy mode (no "use strict"):
var let = 5;        // Technically works (terrible idea)
var static = true;  // Works but confusing

// In strict mode:
"use strict";
var let = 5;        // SyntaxError: Unexpected strict mode reserved word
```

#### Future Reserved Words (Always Forbidden as Identifiers)
```
enum      // Reserved for potential future Enum support
```

#### Special Global Values (Not Keywords, But Don't Override)
These are **not reserved words** — they are properties of the global object. Technically you can create local variables with these names (shadowing the global), but doing so is a dangerous mistake:

```javascript
undefined   // The primitive undefined value
NaN         // Not a Number (result of failed numeric conversions)
Infinity    // Positive infinity (1/0)
```

```javascript
// These are GLOBAL PROPERTIES, not keywords:
console.log(typeof undefined);  // "undefined"
console.log(1 / 0);            // Infinity
console.log("hello" * 2);      // NaN

// DANGEROUS — shadowing globals (don't ever do this!):
function brokenFunc() {
  var undefined = "I'm not undefined!";   // Legal but catastrophic
  var NaN = 42;                           // Legal but catastrophic
  console.log(undefined);  // "I'm not undefined!"
  console.log(NaN);        // 42
}
// Note: In strict mode + modern engines, undefined cannot be reassigned at global scope
```

#### Contextual Keywords (Soft Keywords)
Some words are only keywords in specific contexts — they're valid identifiers elsewhere:
```javascript
async   // Only a keyword before function/arrow function
from    // Only a keyword in: import x from '...'
of      // Only a keyword in: for (x of iterable)
as      // Only a keyword in: import x as y / export x as y
```

```javascript
// These are valid identifiers in other contexts:
const async = "some value";   // Valid (confusing, but not a SyntaxError)
const from = 10;              // Valid
const of = "preposition";     // Valid
```

---

## 7.2 Statements vs. Expressions

This is one of the most fundamental distinctions in JavaScript's grammar — and one that trips up many developers coming from other languages.

### What is a Statement?

A **statement** is an instruction that **performs an action** or **controls the flow of execution**. Statements are the "sentences" of a program — they tell the computer what to do.

**Key characteristic**: Statements **do not produce a value** — they execute for their side effects.

```javascript
// Declaration statements:
let x = 10;
const name = "Vinay";
var oldStyle = true;

// Expression statements (expressions used as statements):
console.log("Hello");   // A function call expression, used as a statement
x = 20;                 // An assignment expression, used as a statement

// Control flow statements:
if (x > 5) {
  console.log("big");
}

for (let i = 0; i < 3; i++) {
  console.log(i);
}

while (x > 0) {
  x--;
}

// Jump statements:
return x;
break;
continue;
throw new Error("oops");
```

### What is an Expression?

An **expression** is any valid unit of code that **evaluates to (produces) a value**. Expressions can be simple or complex, but they always result in a value.

**Key characteristic**: Expressions **always produce a value** — they can be used wherever a value is expected.

```javascript
// Literal expressions (values themselves):
42              // evaluates to: 42
"hello"         // evaluates to: "hello"
true            // evaluates to: true
null            // evaluates to: null

// Arithmetic expressions:
10 + 5          // evaluates to: 15
2 ** 8          // evaluates to: 256
10 % 3          // evaluates to: 1

// String expressions:
"Hello" + " " + "World"  // evaluates to: "Hello World"
`Hello, ${name}!`        // evaluates to: "Hello, Vinay!" (template literal)

// Comparison expressions:
10 > 5          // evaluates to: true
"a" === "a"     // evaluates to: true
x !== null      // evaluates to: true or false

// Logical expressions:
true && false   // evaluates to: false
null || "default"  // evaluates to: "default"
!true           // evaluates to: false

// Assignment expressions (also side-effectful):
x = 10          // evaluates to: 10 (the assigned value)
x += 5          // evaluates to: 15 (the new value of x)

// Function call expressions:
Math.max(1, 5)      // evaluates to: 5
"hello".toUpperCase()  // evaluates to: "HELLO"
parseInt("42px")       // evaluates to: 42

// Object / Array expressions:
{ name: "Vinay" }      // evaluates to: an object
[1, 2, 3]              // evaluates to: an array

// Conditional (ternary) expression:
x > 0 ? "positive" : "non-positive"  // evaluates to: one of the two strings

// Function expression:
function(x) { return x * 2; }        // evaluates to: a function object
(x) => x * 2                         // evaluates to: a function object (arrow)
```

### The Critical Difference: Statements vs. Expressions

| Property | Statement | Expression |
|---|---|---|
| **Produces a value?** | ❌ No | ✅ Yes — always |
| **Can be on the right side of `=`?** | ❌ No | ✅ Yes |
| **Can be passed as an argument?** | ❌ No | ✅ Yes |
| **Can stand alone?** | ✅ Yes | ✅ Yes (as expression statement) |
| **Has side effects?** | Usually yes | Sometimes (function calls, assignment) |
| **Examples** | `if`, `for`, `let x`, `return` | `1+1`, `fn()`, `x=5`, `true` |

```javascript
// You can use an EXPRESSION anywhere a value is expected:
const result = 10 > 5 ? "yes" : "no";  // Ternary expression as value ✅
console.log(2 + 2);                     // Arithmetic expression as argument ✅
const double = (x) => x * 2;           // Function expression as value ✅

// You CANNOT use a STATEMENT where a value is expected:
const result = if (true) { "yes" } else { "no" };  // SyntaxError ❌
console.log(let x = 5);                             // SyntaxError ❌
```

### Expression Statements — The Bridge

Every **expression** can be turned into a **statement** simply by following it with a semicolon (or a newline — thanks to ASI). These are called **expression statements**:

```javascript
// These are all expressions used AS statements:
x = 10;                // assignment expression → expression statement
console.log("hi");     // function call expression → expression statement
x++;                   // increment expression → expression statement
"wasted string";       // string literal expression → expression statement (but useless)
2 + 2;                 // arithmetic expression → expression statement (but useless)
```

> The reverse is **not** true — statements cannot be used as expressions. This is why JavaScript needs the ternary operator (`? :`) — you can't use `if/else` as an expression to produce a value.

### Practical Importance: The Statement vs. Expression Distinction

#### In Conditional Rendering (React/Vue)
```javascript
// JSX / template expressions require EXPRESSIONS, not statements:

// ✅ Correct — ternary is an expression:
return <div>{isLoggedIn ? <UserPanel /> : <LoginButton />}</div>;

// ❌ Wrong — if/else is a statement, cannot be inline:
return <div>{if (isLoggedIn) { <UserPanel /> } else { <LoginButton /> }}</div>;
```

#### In Arrow Functions
```javascript
// Arrow function with expression body (implicit return):
const double = x => x * 2;         // x * 2 is an expression — returned automatically

// Arrow function with statement body (explicit return needed):
const double = x => { return x * 2; };   // {} creates a statement block
const double = x => { x * 2; };          // BUG: returns undefined! Statement, not expression
```

#### In Template Literals
```javascript
const name = "Vinay";

// ✅ Only expressions work inside ${}:
console.log(`Hello, ${name.toUpperCase()}!`);   // function call expression ✅
console.log(`2 + 2 = ${2 + 2}`);               // arithmetic expression ✅

// ❌ Statements don't work inside ${}:
console.log(`Hello, ${let x = 5}`);            // SyntaxError ❌
console.log(`${if (true) { "yes" }}`);         // SyntaxError ❌
```

---

## Summary of Topic 7

```
IDENTIFIERS, STATEMENTS, EXPRESSIONS & GRAMMAR

7.1 IDENTIFIERS & KEYWORDS
  │
  ├── IDENTIFIERS: Names for variables, functions, classes, params, props
  │     ├── Rules: Start with letter/_/$, then letters/digits/_/$
  │     ├── Case-sensitive: name ≠ Name ≠ NAME
  │     └── Conventions: camelCase, PascalCase, SCREAMING_SNAKE, _private
  │
  ├── RESERVED KEYWORDS (cannot be used as identifiers)
  │     ├── Declarations: var, let, const
  │     ├── Functions/Classes: function, class, new, return, this, super
  │     ├── Control flow: if, else, switch, for, while, break, continue
  │     ├── Error handling: try, catch, finally, throw
  │     ├── Modules: import, export
  │     └── Other: delete, typeof, instanceof, in, void, async, await, yield
  │
  ├── LITERAL VALUES (special reserved literals)
  │     └── true, false, null
  │
  └── DISALLOWED / SPECIAL WORDS
        ├── Strict mode reserved: implements, interface, package, private,
        │   protected, public, static, yield
        ├── Future reserved: enum
        ├── Global properties (don't shadow!): undefined, NaN, Infinity
        └── Contextual/soft keywords: async, from, of, as
              (valid identifiers in non-keyword positions)

7.2 STATEMENTS vs. EXPRESSIONS
  │
  ├── STATEMENT: Instruction that performs an action / controls flow
  │     ├── Does NOT produce a value
  │     ├── Cannot be used where a value is expected
  │     └── Examples: if, for, while, let x = 5, return, break
  │
  ├── EXPRESSION: Code unit that evaluates to a value
  │     ├── ALWAYS produces a value
  │     ├── Can be used anywhere a value is expected (right of =, arguments)
  │     └── Examples: 1+1, "hello", fn(), x=5, true, x > 0 ? a : b
  │
  ├── EXPRESSION STATEMENT: Expression used as a statement (add ; or newline)
  │     └── console.log("hi");  x = 10;  x++;
  │
  └── KEY PRACTICAL IMPACTS
        ├── JSX / Templates: Only expressions work inline (${...}, ternary)
        ├── Arrow functions: Expression body → implicit return
        │                    Statement body {} → explicit return required
        └── if/else is a STATEMENT → use ternary (?) for inline conditional values
```

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
