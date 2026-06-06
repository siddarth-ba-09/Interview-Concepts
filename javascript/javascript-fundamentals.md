# JavaScript — Complete Fundamentals Guide

> Perspective: Java developer learning JS. Key differences from Java are highlighted.

---

## Table of Contents

1. [How JavaScript Works](#1-how-javascript-works)
2. [Variables & Scoping](#2-variables--scoping)
3. [Data Types](#3-data-types)
4. [Type Coercion & Equality](#4-type-coercion--equality)
5. [Functions](#5-functions)
6. [Closures](#6-closures)
7. [this Keyword](#7-this-keyword)
8. [Prototypes & Prototype Chain](#8-prototypes--prototype-chain)
9. [Classes (ES6)](#9-classes-es6)
10. [Arrays & Array Methods](#10-arrays--array-methods)
11. [Objects](#11-objects)
12. [Destructuring, Spread & Rest](#12-destructuring-spread--rest)
13. [ES6+ Features](#13-es6-features)
14. [Asynchronous JavaScript](#14-asynchronous-javascript)
15. [Event Loop & Concurrency Model](#15-event-loop--concurrency-model)
16. [Modules](#16-modules)
17. [Error Handling](#17-error-handling)
18. [DOM Manipulation (Basics)](#18-dom-manipulation-basics)
19. [Miscellaneous Concepts](#19-miscellaneous-concepts)
20. [Interview Q&A](#20-interview-qa)

---

## 1. How JavaScript Works

- **Interpreted language** — runs in a JS engine (V8 in Chrome/Node, SpiderMonkey in Firefox).
- **Single-threaded** — one call stack, one thing at a time.
- **Dynamically typed** — types are determined at runtime (unlike Java).
- **JIT compiled** — V8 compiles JS to machine code at runtime for performance.

### Execution Context
Every time JS runs code, it creates an **Execution Context**:
1. **Global Execution Context** — created first, has `window` (browser) or `global` (Node).
2. **Function Execution Context** — created each time a function is called.

Each context has two phases:
- **Memory/Creation Phase** — variables are hoisted, `this` is determined.
- **Execution Phase** — code runs line by line.

---

## 2. Variables & Scoping

### var vs let vs const

| Feature | `var` | `let` | `const` |
|---------|-------|-------|---------|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Hoisting | Hoisted & initialized to `undefined` | Hoisted but NOT initialized (TDZ) | Hoisted but NOT initialized (TDZ) |
| Re-declaration | Allowed | Not allowed | Not allowed |
| Re-assignment | Allowed | Allowed | Not allowed |
| Global object property | Yes (`window.x`) | No | No |

```js
// var — function scoped
function test() {
  if (true) {
    var x = 10; // accessible throughout function
  }
  console.log(x); // 10
}

// let — block scoped
function test2() {
  if (true) {
    let y = 20; // only accessible inside this block
  }
  console.log(y); // ReferenceError
}

// const — must be initialized, cannot be reassigned
const PI = 3.14;
PI = 3.15; // TypeError
const obj = { a: 1 };
obj.a = 2; // ALLOWED — object itself is not reassigned, only property
```

### Hoisting

```js
console.log(a); // undefined (var is hoisted)
var a = 5;

console.log(b); // ReferenceError: Cannot access 'b' before initialization (TDZ)
let b = 5;
```

**Temporal Dead Zone (TDZ):** The period between hoisting of `let`/`const` and their actual declaration line.

### Scope Chain
When JS looks up a variable, it starts from the current scope and moves outward to the global scope — this is the **scope chain**.

---

## 3. Data Types

### Primitive Types (7)
```js
let str = "hello";           // string
let num = 42;                // number (no int/float distinction)
let big = 9007199254740991n; // BigInt
let bool = true;             // boolean
let nothing = null;          // null (intentional absence)
let undef = undefined;       // undefined (unintentionally missing)
let sym = Symbol("id");      // Symbol (unique identifier)
```

### Reference Types
```js
let arr = [1, 2, 3];         // Array
let obj = { name: "Alice" }; // Object
let fn = function() {};      // Function
```

### typeof operator
```js
typeof "hello"    // "string"
typeof 42         // "number"
typeof true       // "boolean"
typeof undefined  // "undefined"
typeof null       // "object"  ← famous bug in JS
typeof {}         // "object"
typeof []         // "object"
typeof function(){} // "function"
```

### Primitives are passed by value; Objects are passed by reference
```js
let a = 5;
let b = a;
b = 10;
console.log(a); // 5 — not affected

let obj1 = { x: 1 };
let obj2 = obj1; // both point to same object
obj2.x = 99;
console.log(obj1.x); // 99 — affected
```

---

## 4. Type Coercion & Equality

### Implicit Coercion (JS tries to convert types)
```js
"5" + 3     // "53"  — number coerced to string
"5" - 3     // 2     — string coerced to number
true + 1    // 2
false + 1   // 1
null + 1    // 1
undefined + 1 // NaN
```

### == vs ===
- `==` (loose equality) — performs type coercion before comparison.
- `===` (strict equality) — no coercion, types must match.

```js
0 == false   // true
0 === false  // false
"" == false  // true
null == undefined  // true
null === undefined // false
NaN == NaN   // false (NaN is not equal to itself!)
```

**Rule of thumb:** Always use `===` to avoid coercion surprises.

---

## 5. Functions

### Function Declaration
```js
function greet(name) {
  return "Hello, " + name;
}
// Hoisted — can be called before declaration
```

### Function Expression
```js
const greet = function(name) {
  return "Hello, " + name;
};
// NOT hoisted
```

### Arrow Functions (ES6)
```js
const greet = (name) => `Hello, ${name}`;
const square = x => x * x;      // single param — no parens needed
const add = (a, b) => a + b;
const getObj = () => ({ key: "val" }); // returning object — wrap in ()

// Arrow functions do NOT have their own `this`
```

### Default Parameters
```js
function greet(name = "World") {
  return `Hello, ${name}`;
}
greet();         // "Hello, World"
greet("Alice");  // "Hello, Alice"
```

### Rest Parameters
```js
function sum(...numbers) {  // collects all args into array
  return numbers.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4); // 10
```

### Arguments Object (not available in arrow functions)
```js
function old() {
  console.log(arguments); // array-like object of all args
}
```

### IIFE (Immediately Invoked Function Expression)
```js
(function() {
  console.log("Runs immediately");
})();
```

### Higher-Order Functions
Functions that take functions as arguments or return functions.
```js
function applyTwice(fn, value) {
  return fn(fn(value));
}
applyTwice(x => x * 2, 3); // 12
```

---

## 6. Closures

A **closure** is a function that remembers the variables from its outer scope even after the outer function has returned.

```js
function makeCounter() {
  let count = 0;  // this variable is "closed over"
  return function() {
    count++;
    return count;
  };
}

const counter = makeCounter();
counter(); // 1
counter(); // 2
counter(); // 3
// count is not accessible from outside, but the inner function remembers it
```

### Practical Uses
- **Data encapsulation / private variables**
- **Factory functions**
- **Memoization**
- **Callbacks and event handlers**

```js
// Common interview trap — var in loops
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3  (var is function-scoped, all share same i)

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2  (let is block-scoped, each iteration has its own i)
```

---

## 7. this Keyword

`this` refers to the **context in which a function is called**, not where it is defined (except arrow functions).

| Context | Value of `this` |
|---------|----------------|
| Global scope (non-strict) | `window` (browser) / `global` (Node) |
| Global scope (strict mode) | `undefined` |
| Regular function call | `window` / `undefined` |
| Method call (`obj.fn()`) | `obj` |
| Arrow function | Inherits `this` from enclosing scope |
| Event handler | The DOM element |
| `new` call | The newly created object |

```js
const person = {
  name: "Alice",
  greet: function() {
    console.log(this.name); // "Alice" — this = person
  },
  greetArrow: () => {
    console.log(this.name); // undefined — arrow inherits outer this (global)
  }
};

// Explicit binding
function sayHi() { console.log(this.name); }
sayHi.call({ name: "Bob" });    // "Bob"
sayHi.apply({ name: "Charlie"}); // "Charlie"
const bound = sayHi.bind({ name: "Dave" });
bound(); // "Dave"
```

---

## 8. Prototypes & Prototype Chain

Every JS object has a hidden `[[Prototype]]` property pointing to another object. When you access a property, JS looks up the prototype chain.

```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function() {
  return `${this.name} makes a sound`;
};

const dog = new Animal("Dog");
dog.speak(); // "Dog makes a sound"
// dog.__proto__ === Animal.prototype  → true
// Animal.prototype.__proto__ === Object.prototype  → true
// Object.prototype.__proto__ === null
```

### Prototypal Inheritance
```js
function Dog(name) {
  Animal.call(this, name); // call parent constructor
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function() { return "Woof!"; };
```

> This is the old way. ES6 Classes (next section) are syntactic sugar over this.

---

## 9. Classes (ES6)

```js
class Animal {
  #sound; // private field (ES2022)

  constructor(name, sound) {
    this.name = name;
    this.#sound = sound;
  }

  speak() {
    return `${this.name} says ${this.#sound}`;
  }

  static create(name) {  // static method
    return new Animal(name, "...");
  }

  get info() {           // getter
    return `Name: ${this.name}`;
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name, "Woof");  // must call super before using this
  }

  fetch() {
    return `${this.name} fetches!`;
  }
}

const d = new Dog("Rex");
d.speak();  // "Rex says Woof"
d instanceof Dog;     // true
d instanceof Animal;  // true
```

> **Note:** JS classes are syntactic sugar over prototypes. Under the hood, `Dog.prototype.__proto__ === Animal.prototype`.

---

## 10. Arrays & Array Methods

### Creation
```js
const arr = [1, 2, 3];
const arr2 = new Array(3); // [empty × 3]
const arr3 = Array.from("hello"); // ['h','e','l','l','o']
const arr4 = Array.from({length: 5}, (_, i) => i); // [0,1,2,3,4]
```

### Essential Methods

| Method | Description | Returns | Mutates? |
|--------|-------------|---------|----------|
| `push(...items)` | Add to end | new length | Yes |
| `pop()` | Remove from end | removed item | Yes |
| `shift()` | Remove from start | removed item | Yes |
| `unshift(...items)` | Add to start | new length | Yes |
| `splice(i, n, ...items)` | Remove/insert at index | removed items | Yes |
| `slice(start, end)` | Extract portion | new array | No |
| `concat(...arrays)` | Merge arrays | new array | No |
| `indexOf(item)` | Find index | index or -1 | No |
| `includes(item)` | Check existence | boolean | No |
| `find(fn)` | First matching item | item or undefined | No |
| `findIndex(fn)` | Index of first match | index or -1 | No |
| `filter(fn)` | Filter items | new array | No |
| `map(fn)` | Transform items | new array | No |
| `reduce(fn, init)` | Accumulate to single value | value | No |
| `forEach(fn)` | Iterate (no return) | undefined | No |
| `sort(fn)` | Sort in-place | same array | Yes |
| `reverse()` | Reverse in-place | same array | Yes |
| `flat(depth)` | Flatten nested arrays | new array | No |
| `flatMap(fn)` | map + flat(1) | new array | No |
| `every(fn)` | All items match? | boolean | No |
| `some(fn)` | At least one matches? | boolean | No |
| `join(sep)` | Join to string | string | No |
| `fill(val, s, e)` | Fill with value | same array | Yes |

```js
const nums = [1, 2, 3, 4, 5];

nums.map(x => x * 2);           // [2, 4, 6, 8, 10]
nums.filter(x => x % 2 === 0);  // [2, 4]
nums.reduce((acc, x) => acc + x, 0); // 15
nums.find(x => x > 3);          // 4
nums.every(x => x > 0);         // true
nums.some(x => x > 4);          // true

// Chaining
const result = [1,2,3,4,5,6]
  .filter(x => x % 2 === 0)
  .map(x => x * 10)
  .reduce((acc, x) => acc + x, 0);
// 120
```

---

## 11. Objects

### Creation
```js
const obj1 = { name: "Alice", age: 30 }; // object literal

const obj2 = Object.create(null); // no prototype

function Person(name) { this.name = name; }
const obj3 = new Person("Bob");
```

### Property Access
```js
obj.name        // dot notation
obj["name"]     // bracket notation (useful for dynamic keys)
const key = "name";
obj[key]        // dynamic key access
```

### Shorthand & Computed Properties (ES6)
```js
const name = "Alice";
const age = 30;
const person = { name, age }; // shorthand — { name: name, age: age }

const key = "dynamic";
const obj = { [key]: "value" }; // { dynamic: "value" }
```

### Object Methods
```js
Object.keys(obj)         // array of keys
Object.values(obj)       // array of values
Object.entries(obj)      // array of [key, value] pairs
Object.assign(target, src) // shallow merge/copy
Object.freeze(obj)       // makes obj immutable
Object.create(proto)     // creates obj with given prototype
const copy = { ...obj }  // spread — shallow copy
```

### Property Descriptors
```js
Object.defineProperty(obj, "id", {
  value: 42,
  writable: false,
  enumerable: false,
  configurable: false
});
```

---

## 12. Destructuring, Spread & Rest

### Array Destructuring
```js
const [a, b, c] = [1, 2, 3];
const [first, , third] = [1, 2, 3]; // skip elements
const [x, ...rest] = [1, 2, 3, 4];  // x=1, rest=[2,3,4]
const [p = 10, q = 20] = [5];       // defaults: p=5, q=20
```

### Object Destructuring
```js
const { name, age } = person;
const { name: fullName } = person;          // rename
const { city = "Unknown" } = person;        // default value
const { address: { street } } = person;     // nested
function greet({ name, age = 0 }) { ... }   // in function params
```

### Spread Operator
```js
const arr1 = [1, 2];
const arr2 = [3, 4];
const combined = [...arr1, ...arr2]; // [1,2,3,4]

const obj1 = { a: 1 };
const obj2 = { b: 2 };
const merged = { ...obj1, ...obj2 }; // { a:1, b:2 }

// Function call
Math.max(...[1, 2, 3]); // 3
```

### Rest Parameters (different from spread — used in function definition)
```js
function sum(first, ...others) {
  return others.reduce((a, b) => a + b, first);
}
```

---

## 13. ES6+ Features

### Template Literals
```js
const name = "Alice";
const msg = `Hello, ${name}! ${1 + 1} + 1 = ${1 + 2}`; // tagged templates also exist
```

### Optional Chaining (?.)
```js
const city = user?.address?.city; // no error if address is null/undefined
const first = arr?.[0];           // safe array access
fn?.();                           // safe function call
```

### Nullish Coalescing (??)
```js
const name = user.name ?? "Anonymous"; // only falls back on null/undefined (not 0, "", false)
// vs. || which falls back on any falsy value
const port = config.port || 3000;      // 0 would fallback (bug!)
const port2 = config.port ?? 3000;     // 0 is kept (correct)
```

### Logical Assignment Operators
```js
a ||= b;  // a = a || b
a &&= b;  // a = a && b
a ??= b;  // a = a ?? b
```

### Symbol
```js
const id = Symbol("id");  // always unique
const id2 = Symbol("id");
id === id2; // false
// Used as object keys that won't conflict
```

### Iterators & Generators
```js
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i; // pauses and returns value
  }
}

const gen = range(1, 3);
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true }

for (const num of range(1, 5)) { ... }
```

### Map and Set
```js
// Map — key-value, any type as key (unlike plain objects)
const map = new Map();
map.set("key", "value");
map.set(42, "number key");
map.get("key");     // "value"
map.has("key");     // true
map.size;           // 2
map.delete("key");
for (const [k, v] of map) { ... }

// Set — unique values
const set = new Set([1, 2, 2, 3, 3]);
set.size; // 3
set.add(4);
set.has(2); // true
set.delete(1);
const unique = [...new Set([1,1,2,2,3])]; // [1,2,3]
```

### WeakMap & WeakSet
- Keys/values are weakly referenced — garbage collected when no other references.
- Keys must be objects. No size property, not iterable.
- Use case: metadata storage without memory leaks.

---

## 14. Asynchronous JavaScript

### Callbacks
```js
function fetchData(callback) {
  setTimeout(() => callback(null, "data"), 1000);
}
fetchData((err, data) => {
  if (err) { /* handle */ }
  console.log(data);
});
// Problem: callback hell / pyramid of doom
```

### Promises
```js
const promise = new Promise((resolve, reject) => {
  // async operation
  if (success) resolve(value);
  else reject(error);
});

promise
  .then(value => { /* success */ })
  .catch(error => { /* failure */ })
  .finally(() => { /* always runs */ });

// Promise combinators
Promise.all([p1, p2, p3])       // waits for ALL, rejects if any fails
Promise.allSettled([p1, p2])    // waits for ALL, never rejects
Promise.race([p1, p2])          // first to settle (resolve or reject)
Promise.any([p1, p2])           // first to RESOLVE (ignores rejections)
Promise.resolve(value)          // creates resolved promise
Promise.reject(error)           // creates rejected promise
```

### Async/Await (ES2017)
```js
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`); // pauses until promise resolves
    const user = await response.json();
    return user;
  } catch (error) {
    console.error("Error:", error);
    throw error;
  }
}

// async function always returns a Promise
fetchUser(1).then(user => console.log(user));

// Parallel execution
const [user, posts] = await Promise.all([
  fetchUser(1),
  fetchPosts(1)
]);
```

### Key Concepts
- `async` functions always return a Promise.
- `await` can only be used inside `async` functions (or top-level modules).
- `await` does NOT block the thread — it pauses the async function and lets other code run.

---

## 15. Event Loop & Concurrency Model

JS is single-threaded but non-blocking thanks to the event loop.

```
         ┌─────────────────┐
         │   Call Stack     │  ← where code executes (LIFO)
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │  Web APIs / C++ │  ← setTimeout, fetch, DOM events
         └────────┬────────┘
                  │
         ┌────────▼────────┐   ┌─────────────────────┐
         │  Callback Queue  │   │  Microtask Queue     │
         │  (Macro tasks)   │   │  (Promise callbacks) │
         │  setTimeout      │   │  queueMicrotask      │
         │  setInterval     │   │  MutationObserver    │
         └────────┬────────┘   └──────────┬──────────┘
                  │                       │
         ┌────────▼───────────────────────▼────────┐
         │              Event Loop                  │
         │  Checks if Call Stack is empty,          │
         │  picks tasks: Microtasks first, then     │
         │  Macrotasks one at a time                │
         └──────────────────────────────────────────┘
```

**Priority:** Call Stack → Microtask Queue → Macrotask Queue

```js
console.log("1");
setTimeout(() => console.log("2"), 0); // macrotask
Promise.resolve().then(() => console.log("3")); // microtask
console.log("4");

// Output: 1, 4, 3, 2
// Reason: sync (1,4) → microtask (3) → macrotask (2)
```

---

## 16. Modules

### CommonJS (Node.js)
```js
// export
module.exports = { greet, PI };
module.exports.name = "Alice";

// import
const { greet } = require("./module");
const express = require("express");
```

### ES Modules (ESM — browser & modern Node.js)
```js
// Named exports
export const PI = 3.14;
export function greet() {}
export class Animal {}

// Default export (one per file)
export default function main() {}

// Import
import main from "./main.js";
import { PI, greet } from "./math.js";
import * as math from "./math.js"; // namespace import
import { greet as hello } from "./greet.js"; // rename

// Dynamic import
const module = await import("./module.js");
```

### CommonJS vs ESM

| Feature | CommonJS | ESM |
|---------|----------|-----|
| Syntax | `require/module.exports` | `import/export` |
| Loading | Synchronous | Asynchronous |
| Tree shaking | No | Yes |
| Top-level await | No | Yes |
| `__dirname`, `__filename` | Available | Not available (use `import.meta.url`) |

---

## 17. Error Handling

```js
// try-catch-finally
try {
  const result = riskyOperation();
} catch (error) {
  if (error instanceof TypeError) {
    console.error("Type error:", error.message);
  } else {
    throw error; // re-throw unknown errors
  }
} finally {
  cleanup(); // always runs
}

// Custom Error
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

throw new ValidationError("Invalid email", "email");

// Async error handling
async function fetchData() {
  try {
    const data = await fetch("/api/data");
    if (!data.ok) throw new Error(`HTTP ${data.status}`);
    return data.json();
  } catch (err) {
    console.error(err);
  }
}

// Unhandled promise rejections
process.on("unhandledRejection", (reason) => { /* Node */ });
window.addEventListener("unhandledrejection", (e) => { /* Browser */ });
```

---

## 18. DOM Manipulation (Basics)

```js
// Selecting elements
document.getElementById("id");
document.querySelector(".class");      // first match
document.querySelectorAll("div.card"); // all matches (NodeList)
document.getElementsByClassName("cls"); // HTMLCollection

// Modifying
element.textContent = "new text";       // plain text
element.innerHTML = "<b>bold</b>";      // HTML (XSS risk!)
element.setAttribute("href", "/path");
element.classList.add("active");
element.classList.remove("hidden");
element.classList.toggle("open");
element.style.color = "red";

// Creating & inserting
const div = document.createElement("div");
div.textContent = "Hello";
parent.appendChild(div);
parent.insertBefore(div, referenceNode);
parent.removeChild(div);

// Events
button.addEventListener("click", (event) => {
  event.preventDefault();   // stop default behavior
  event.stopPropagation();  // stop bubbling
  console.log(event.target);
});

// Event Delegation — attach one listener to parent
document.getElementById("list").addEventListener("click", (e) => {
  if (e.target.matches("li")) {
    console.log("Clicked:", e.target.textContent);
  }
});
```

---

## 19. Miscellaneous Concepts

### Truthy & Falsy
```js
// Falsy values (6 total)
false, 0, -0, 0n, "", '', ``, null, undefined, NaN

// Everything else is truthy
// Tricky truthy: "0", [], {}, "false"
```

### Short-circuit Evaluation
```js
const name = user && user.name;          // if user is truthy, get name
const display = name || "Anonymous";     // fallback
const val = config.timeout ?? 3000;      // nullish coalescing
```

### JSON
```js
JSON.stringify({ a: 1, b: [2, 3] }); // → string
JSON.parse('{"a":1}');               // → object
JSON.stringify(obj, null, 2);        // pretty print (2 spaces)
```

### Memoization
```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

### Currying
```js
const add = a => b => c => a + b + c;
add(1)(2)(3); // 6
```

### Debounce & Throttle
```js
// Debounce — delays execution until after N ms of no calls
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle — executes at most once per N ms
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}
```

### Proxy & Reflect
```js
const handler = {
  get(target, prop) {
    return prop in target ? target[prop] : `Property ${prop} not found`;
  },
  set(target, prop, value) {
    if (typeof value !== "number") throw new TypeError("Only numbers!");
    target[prop] = value;
    return true;
  }
};
const proxy = new Proxy({}, handler);
```

### Symbols as Iteration Protocol
```js
class Range {
  constructor(start, end) { this.start = start; this.end = end; }
  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return { next() { return current <= end ? { value: current++, done: false } : { done: true }; }};
  }
}
[...new Range(1, 5)]; // [1, 2, 3, 4, 5]
```

---

## 20. Interview Q&A

**Q: What is the difference between `null` and `undefined`?**
> `undefined` means a variable has been declared but not assigned a value. `null` is an explicit assignment indicating "no value". `typeof null === "object"` is a known JS bug.

**Q: Explain closure with an example.**
> A closure is a function that retains access to its lexical scope even when executed outside that scope. Example: a counter factory where the inner function remembers `count`.

**Q: What is the event loop?**
> JS is single-threaded. The event loop continuously checks if the call stack is empty and pushes callbacks from the task queues (microtasks first, then macrotasks) onto the stack.

**Q: What is the difference between `==` and `===`?**
> `==` performs type coercion before comparison; `===` does strict comparison without coercion.

**Q: What are Promises and how do they differ from callbacks?**
> Promises are objects representing eventual completion/failure of async operations. They offer chaining (`.then/.catch`), better error propagation, and avoid callback hell.

**Q: What is `Promise.all` vs `Promise.allSettled`?**
> `Promise.all` rejects immediately if any promise rejects. `Promise.allSettled` always waits for all and returns an array of `{status, value/reason}` objects.

**Q: What is hoisting?**
> JS moves variable and function declarations to the top of their scope before execution. `var` is hoisted and initialized to `undefined`. `let`/`const` are hoisted but remain in TDZ until their declaration line.

**Q: What is the difference between `map` and `forEach`?**
> `map` returns a new array with transformed values. `forEach` returns `undefined` and is only used for side effects.

**Q: Explain `call`, `apply`, and `bind`.**
> All three set `this` explicitly. `call(ctx, arg1, arg2)` calls immediately. `apply(ctx, [args])` calls immediately with array. `bind(ctx)` returns a new function with `this` bound permanently.

**Q: What is event delegation?**
> Attaching a single event listener to a parent element instead of individual children, using `event.target` to identify the actual clicked element. More performant for dynamic lists.
