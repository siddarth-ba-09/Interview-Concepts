# TypeScript — Complete Fundamentals Guide

> TypeScript is a **typed superset of JavaScript** that compiles to plain JS. As a Java developer, many concepts (generics, interfaces, enums) will feel familiar.

---

## Table of Contents

1. [Why TypeScript](#1-why-typescript)
2. [Setup & tsconfig.json](#2-setup--tsconfigjson)
3. [Basic Types](#3-basic-types)
4. [Type Annotations & Inference](#4-type-annotations--inference)
5. [Interfaces](#5-interfaces)
6. [Type Aliases](#6-type-aliases)
7. [Interface vs Type Alias](#7-interface-vs-type-alias)
8. [Union & Intersection Types](#8-union--intersection-types)
9. [Enums](#9-enums)
10. [Generics](#10-generics)
11. [Classes](#11-classes)
12. [Functions in TypeScript](#12-functions-in-typescript)
13. [Type Guards & Narrowing](#13-type-guards--narrowing)
14. [Utility Types](#14-utility-types)
15. [Mapped Types](#15-mapped-types)
16. [Conditional Types](#16-conditional-types)
17. [Decorators](#17-decorators)
18. [Modules & Declaration Files](#18-modules--declaration-files)
19. [Strict Mode](#19-strict-mode)
20. [Miscellaneous Concepts](#20-miscellaneous-concepts)
21. [Interview Q&A](#21-interview-qa)

---

## 1. Why TypeScript

| JavaScript | TypeScript |
|-----------|-----------|
| Dynamic types (runtime errors) | Static types (compile-time errors) |
| No IntelliSense hints | Rich IDE support |
| No compile step | Compiles to JS |
| Flexible but error-prone | Verbose but safe |

- **TypeScript is compiled away** — the browser/Node runs plain JavaScript.
- TS adds: type annotations, interfaces, generics, enums, decorators, access modifiers.
- 100% backward compatible — any valid JS is valid TS.

---

## 2. Setup & tsconfig.json

```bash
npm install -g typescript
tsc --init       # generates tsconfig.json
tsc              # compiles all .ts files
tsc --watch      # watch mode
npx ts-node file.ts  # run TS directly
```

### Key tsconfig.json Options
```json
{
  "compilerOptions": {
    "target": "ES2020",         // JS version to compile to
    "module": "commonjs",       // module system (commonjs, ESNext)
    "lib": ["ES2020", "DOM"],   // built-in type definitions to include
    "outDir": "./dist",         // output directory
    "rootDir": "./src",         // input directory
    "strict": true,             // enable all strict checks
    "noImplicitAny": true,      // error on implicit 'any'
    "strictNullChecks": true,   // null/undefined are not assignable to other types
    "esModuleInterop": true,    // cleaner default import interop
    "sourceMap": true,          // generate .map files for debugging
    "declaration": true,        // generate .d.ts files
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

---

## 3. Basic Types

```ts
// Primitives
let name: string = "Alice";
let age: number = 30;
let isActive: boolean = true;
let bigNum: bigint = 100n;
let sym: symbol = Symbol("key");

// Special types
let anything: any = "could be anything"; // opt-out of type checking (avoid!)
let unknown: unknown = getData();         // safer any — must type-check before use
let undef: undefined = undefined;
let nothing: null = null;
let noReturn: void = undefined;          // function that returns nothing
let impossible: never;                   // value that never occurs

// Arrays
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ["a", "b"];
let mixed: (string | number)[] = [1, "a"];

// Tuple — fixed-length, typed array
let pair: [string, number] = ["Alice", 30];
let labeled: [name: string, age: number] = ["Alice", 30]; // named tuple
pair[0].toUpperCase(); // TS knows pair[0] is string

// Object
let user: { name: string; age: number; email?: string } = { name: "Alice", age: 30 };
```

### any vs unknown vs never

| Type | Description | Use Case |
|------|-------------|----------|
| `any` | Completely disables type checking | Legacy code, quick workarounds |
| `unknown` | Like `any` but safe — must narrow before use | User input, dynamic data |
| `never` | Function never returns (throws or infinite loop) | Exhaustive checks, error functions |

```ts
function assertNever(x: never): never {
  throw new Error("Unexpected value: " + x);
}

// unknown — must check before use
function process(value: unknown) {
  if (typeof value === "string") {
    value.toUpperCase(); // OK — narrowed to string
  }
}
```

---

## 4. Type Annotations & Inference

TypeScript can **infer** types — you don't always need to annotate.

```ts
let name = "Alice";   // inferred as string
let num = 42;         // inferred as number
const PI = 3.14;      // inferred as literal type 3.14

// Explicit annotation needed when
let items: string[]; // declaration without initialization
function greet(name: string): string { return `Hello ${name}`; }
```

### Type Assertion (casting)
```ts
const input = document.getElementById("name") as HTMLInputElement;
input.value; // TS now knows it's HTMLInputElement

// Alternative angle-bracket syntax (not valid in JSX)
const input2 = <HTMLInputElement>document.getElementById("name");

// Double assertion for incompatible types (use sparingly)
const value = someValue as unknown as string;
```

### Non-null Assertion Operator (!)
```ts
const element = document.getElementById("id")!; // tell TS "trust me, not null"
element.textContent = "hello"; // no error
```

---

## 5. Interfaces

Define the **shape** of an object.

```ts
interface User {
  readonly id: number;      // cannot be changed after creation
  name: string;
  email?: string;           // optional
  greet(): string;          // method signature
  log: (msg: string) => void; // arrow function method
}

const user: User = {
  id: 1,
  name: "Alice",
  greet() { return `Hi, ${this.name}`; },
  log(msg) { console.log(msg); }
};

// Interface extension (like Java extends)
interface Admin extends User {
  role: string;
  permissions: string[];
}

// Multiple extension
interface SuperAdmin extends Admin, AuditLog {
  superPower: string;
}
```

### Index Signatures
```ts
interface StringMap {
  [key: string]: string; // any number of string → string properties
}

interface NumberMap {
  [index: number]: string; // array-like
}
```

### Interface for Function Types
```ts
interface Transformer {
  (input: string): string;
}
const toUpper: Transformer = (s) => s.toUpperCase();
```

### Declaration Merging (unique to interfaces)
```ts
interface Window {
  myCustomProp: string;
}
// TS merges both declarations into one
```

---

## 6. Type Aliases

```ts
type ID = string | number;
type Coordinates = [number, number];
type Callback = (err: Error | null, result: string) => void;
type User = {
  id: ID;
  name: string;
};

// Template literal types
type EventName = "click" | "focus" | "blur";
type Handler = `on${Capitalize<EventName>}`; // "onClick" | "onFocus" | "onBlur"
```

---

## 7. Interface vs Type Alias

| Feature | `interface` | `type` |
|---------|-------------|--------|
| Extend | `extends` | `&` (intersection) |
| Merge declarations | Yes (automatic) | No (error) |
| Computed properties | No | Yes |
| Primitives/Union/Tuple | No | Yes |
| Implements in class | Yes | Yes (for object shapes) |

**Rule of thumb:** Use `interface` for object shapes (especially public APIs), `type` for unions, intersections, tuples, and complex types.

---

## 8. Union & Intersection Types

### Union ( | ) — OR
```ts
type ID = string | number;
let id: ID = "abc";
id = 123; // also valid

type Status = "pending" | "active" | "inactive"; // literal union (like enum)

function format(value: string | number): string {
  if (typeof value === "string") return value.toUpperCase(); // narrowing
  return value.toFixed(2);
}
```

### Intersection ( & ) — AND (combines types)
```ts
type Timestamped = { createdAt: Date; updatedAt: Date };
type Named = { name: string };

type Entity = Named & Timestamped;
// Entity has: name, createdAt, updatedAt — must have ALL properties

const e: Entity = { name: "Alice", createdAt: new Date(), updatedAt: new Date() };
```

### Discriminated Unions (Tagged Unions)
```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {          // TS narrows each case
    case "circle":   return Math.PI * shape.radius ** 2;
    case "rectangle": return shape.width * shape.height;
    case "triangle":  return 0.5 * shape.base * shape.height;
  }
}
```

---

## 9. Enums

```ts
// Numeric enum (default, starts at 0)
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}
Direction.Up;    // 0
Direction[0];    // "Up" — reverse mapping

// Custom values
enum StatusCode {
  OK = 200,
  NotFound = 404,
  ServerError = 500
}

// String enum (no reverse mapping)
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE"
}

// Const enum — inlined at compile time (no JS object generated)
const enum Flags {
  Read = 1 << 0,   // 1
  Write = 1 << 1,  // 2
  Execute = 1 << 2 // 4
}
```

### Enum vs Union Type
```ts
// Modern preference: use union types for simple enums
type Direction = "up" | "down" | "left" | "right";
// Better tree-shaking, simpler JS output
```

---

## 10. Generics

Generics allow writing reusable, type-safe code. Similar to Java generics.

```ts
// Generic function
function identity<T>(value: T): T {
  return value;
}
identity<string>("hello"); // explicit
identity(42);              // inferred as number

// Generic interface
interface Repository<T> {
  findById(id: number): T;
  findAll(): T[];
  save(entity: T): T;
}

// Generic class
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  isEmpty(): boolean { return this.items.length === 0; }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
numStack.pop(); // 2

// Generic constraints
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
getProperty({ name: "Alice", age: 30 }, "name"); // string
getProperty({ name: "Alice", age: 30 }, "age");  // number
// getProperty({ name: "Alice" }, "email"); // Error!

// Multiple type parameters
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}

// Default generic type
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}
```

### `keyof` and `typeof`
```ts
type UserKeys = keyof User; // "id" | "name" | "email"

const config = { host: "localhost", port: 3000 };
type Config = typeof config; // { host: string; port: number }

// ReturnType utility uses typeof
function createUser() { return { id: 1, name: "Alice" }; }
type UserType = ReturnType<typeof createUser>; // { id: number; name: string }
```

---

## 11. Classes

TypeScript adds access modifiers, readonly, abstract, and implements to JS classes.

```ts
abstract class Animal {
  #id: number;           // private field (JS private)
  private secret: string; // TS private (accessible via JS)
  protected name: string;
  public readonly species: string;

  constructor(id: number, name: string, species: string) {
    this.#id = id;
    this.name = name;
    this.species = species;
    this.secret = "hidden";
  }

  abstract sound(): string; // must be implemented in subclass

  describe(): string {
    return `${this.name} (${this.species})`;
  }
}

interface Trainable {
  train(command: string): boolean;
}

class Dog extends Animal implements Trainable {
  constructor(id: number, name: string) {
    super(id, name, "Canis lupus");
  }

  sound(): string { return "Woof"; }

  train(command: string): boolean {
    console.log(`${this.name} learned: ${command}`);
    return true;
  }
}

// Constructor shorthand (parameter properties)
class Point {
  constructor(
    public readonly x: number,
    public readonly y: number
  ) {} // x and y automatically assigned as properties
}
const p = new Point(1, 2); // p.x = 1, p.y = 2
```

### Access Modifiers Summary

| Modifier | Class | Subclass | Outside |
|----------|-------|----------|---------|
| `public` | ✓ | ✓ | ✓ |
| `protected` | ✓ | ✓ | ✗ |
| `private` | ✓ | ✗ | ✗ |
| `#name` (JS private) | ✓ | ✗ | ✗ (truly private) |
| `readonly` | ✓ (read) | ✓ (read) | ✓ (read) |

---

## 12. Functions in TypeScript

```ts
// Parameter and return types
function add(a: number, b: number): number { return a + b; }

// Optional and default parameters
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}`;
}

// Function overloads
function format(value: string): string;
function format(value: number, decimals: number): string;
function format(value: string | number, decimals?: number): string {
  if (typeof value === "string") return value.toUpperCase();
  return value.toFixed(decimals ?? 2);
}

// Generic arrow function in .tsx files (need trailing comma)
const identity = <T,>(value: T): T => value;

// Function type
type Comparator<T> = (a: T, b: T) => number;
const numSort: Comparator<number> = (a, b) => a - b;

// void vs never
function logMessage(msg: string): void { console.log(msg); } // returns undefined
function throwError(msg: string): never { throw new Error(msg); } // never returns

// this parameter (fake parameter, removed at compile time)
function greetUser(this: { name: string }): string {
  return `Hello ${this.name}`;
}
```

---

## 13. Type Guards & Narrowing

TypeScript narrows types within conditional blocks.

```ts
// typeof guard
function process(value: string | number) {
  if (typeof value === "string") {
    value.toUpperCase(); // string here
  } else {
    value.toFixed(2);    // number here
  }
}

// instanceof guard
function handleError(error: Error | string) {
  if (error instanceof Error) {
    console.log(error.message);
  } else {
    console.log(error);
  }
}

// in operator guard
interface Cat { meow(): void; }
interface Dog { bark(): void; }
function makeSound(animal: Cat | Dog) {
  if ("meow" in animal) {
    animal.meow(); // Cat here
  } else {
    animal.bark(); // Dog here
  }
}

// Custom type guard (type predicate)
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isUser(obj: unknown): obj is User {
  return typeof obj === "object" && obj !== null && "name" in obj && "id" in obj;
}

// Assertion function (throws if condition fails)
function assertIsString(val: unknown): asserts val is string {
  if (typeof val !== "string") throw new Error("Not a string!");
}
```

---

## 14. Utility Types

Built-in generic types for common transformations.

```ts
interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

// Partial<T> — all properties optional
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; role?: string }

// Required<T> — all properties required
type RequiredUser = Required<Partial<User>>;

// Readonly<T> — all properties readonly
type ReadonlyUser = Readonly<User>;

// Pick<T, K> — select subset of properties
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string }

// Omit<T, K> — exclude subset of properties
type UserWithoutId = Omit<User, "id">;
// { name: string; email: string; role: string }

// Record<K, V> — object with keys K and values V
type RoleMap = Record<string, User[]>;
type Config = Record<"host" | "port" | "db", string>;

// Exclude<T, U> — remove types from union
type WithoutString = Exclude<string | number | boolean, string>;
// number | boolean

// Extract<T, U> — keep only types in union
type OnlyString = Extract<string | number | boolean, string | number>;
// string | number

// NonNullable<T> — remove null and undefined
type NotNull = NonNullable<string | null | undefined>;
// string

// ReturnType<T>
function fetchUser(): { id: number; name: string } { return { id: 1, name: "A" }; }
type FetchResult = ReturnType<typeof fetchUser>; // { id: number; name: string }

// Parameters<T>
type FetchParams = Parameters<typeof fetchUser>; // []

// InstanceType<T>
class MyService {}
type ServiceInstance = InstanceType<typeof MyService>; // MyService

// Awaited<T> — unwrap Promise type
type Data = Awaited<Promise<string>>; // string
type NestedData = Awaited<Promise<Promise<number>>>; // number
```

---

## 15. Mapped Types

Transform every property of a type.

```ts
// Making all properties nullable
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Making all properties optional (same as Partial)
type Optional<T> = { [K in keyof T]?: T[K] };

// Making all properties readonly (same as Readonly)
type Immutable<T> = { readonly [K in keyof T]: T[K] };

// Removing readonly
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// Removing optional
type Concrete<T> = { [K in keyof T]-?: T[K] };

// Remapping keys
type RenameId<T> = {
  [K in keyof T as K extends "id" ? "uuid" : K]: T[K]
};

// Filtering properties by type
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]
};

type StringProps = OnlyStrings<{ id: number; name: string; active: boolean }>;
// { name: string }
```

---

## 16. Conditional Types

```ts
// T extends U ? X : Y
type IsString<T> = T extends string ? "yes" : "no";
type A = IsString<string>;  // "yes"
type B = IsString<number>;  // "no"

// Extracting inner type
type Unwrap<T> = T extends Array<infer U> ? U : T;
type NumElement = Unwrap<number[]>;  // number
type SameStr = Unwrap<string>;       // string (not an array)

// Flatten return type of async function
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

// Distributive conditional types
type ToArray<T> = T extends any ? T[] : never;
type StrOrNumArr = ToArray<string | number>; // string[] | number[]
```

---

## 17. Decorators

> Commonly used in Angular (components, services) and NestJS (controllers, injectable).  
> Requires `"experimentalDecorators": true` in tsconfig.

```ts
// Class decorator
function Sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

// Decorator factory (returns decorator)
function Component(options: { selector: string; template: string }) {
  return function(target: Function) {
    target.prototype.selector = options.selector;
    target.prototype.template = options.template;
  };
}

@Sealed
@Component({ selector: "app-root", template: "<div>Hello</div>" })
class AppComponent {
  title = "My App";
}

// Method decorator
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with`, args);
    const result = original.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };
  return descriptor;
}

class MathService {
  @Log
  add(a: number, b: number): number { return a + b; }
}

// Property decorator
function Required(target: any, key: string) {
  let val = target[key];
  Object.defineProperty(target, key, {
    get: () => val,
    set: (newVal) => {
      if (!newVal) throw new Error(`${key} is required`);
      val = newVal;
    }
  });
}

// Parameter decorator
function Validate(target: any, key: string, index: number) {
  // mark parameter for validation
}
```

---

## 18. Modules & Declaration Files

### Declaration Files (.d.ts)
Provide type information for JavaScript libraries.

```ts
// my-lib.d.ts
declare module "my-lib" {
  export function greet(name: string): string;
  export const VERSION: string;
  export interface Options {
    timeout?: number;
  }
}

// Using global declarations
declare const __DEV__: boolean;
declare function require(module: string): any;
```

### DefinitelyTyped
```bash
npm install @types/express @types/node @types/lodash
# Provides type definitions for popular JS libraries
```

### Namespace (for global code organization)
```ts
namespace Utils {
  export function format(date: Date): string { return date.toISOString(); }
  export interface Config { env: string; }
}
Utils.format(new Date());
```

---

## 19. Strict Mode

When `"strict": true`, enables all these:

| Flag | What it does |
|------|-------------|
| `noImplicitAny` | Error when type can't be inferred (not `any`) |
| `strictNullChecks` | `null`/`undefined` not assignable to other types |
| `strictFunctionTypes` | Stricter checking of function types |
| `strictBindCallApply` | Strict checking of `bind`, `call`, `apply` |
| `strictPropertyInitialization` | Properties must be initialized in constructor |
| `noImplicitThis` | `this` must be explicitly typed |
| `alwaysStrict` | Emit `"use strict"` in all files |

**Always use strict mode in new projects.**

---

## 20. Miscellaneous Concepts

### Template Literal Types
```ts
type EventName = "click" | "change" | "blur";
type HandlerName = `on${Capitalize<EventName>}`; // "onClick" | "onChange" | "onBlur"

type CSSProperty = `${string}-${string}`;
type PxValue = `${number}px`;
```

### Satisfies Operator (TS 4.9+)
```ts
type Colors = Record<string, [number, number, number] | string>;
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255]
} satisfies Colors;
// Validates against Colors but preserves literal types
palette.red; // [number, number, number] — not string | [number, number, number]
```

### const Assertions
```ts
const colors = ["red", "green", "blue"] as const;
// readonly ["red", "green", "blue"] — tuple, not string[]

type Color = typeof colors[number]; // "red" | "green" | "blue"

const config = {
  host: "localhost",
  port: 3000
} as const;
// All properties become readonly literals
```

### Indexed Access Types
```ts
type UserName = User["name"];        // string
type UserProps = User["name" | "id"]; // string | number
type ArrayElement = string[][number]; // string
```

### Infer in Conditional Types
```ts
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;
type Arg = FirstArg<(name: string, age: number) => void>; // string
```

### Module Augmentation
```ts
// Extend existing module types
import "express";
declare module "express" {
  interface Request {
    user?: { id: string; role: string };
  }
}
```

---

## 21. Interview Q&A

**Q: What is the difference between `interface` and `type`?**
> Both define shapes. Interfaces support declaration merging and are better for object shapes. Type aliases can represent unions, intersections, primitives, and tuples. Use `interface` for public API shapes, `type` for complex or computed types.

**Q: What is the difference between `any` and `unknown`?**
> `any` disables type checking entirely. `unknown` is the type-safe alternative — you must perform a type check (narrowing) before using the value. Prefer `unknown` for external data.

**Q: What are generics in TypeScript?**
> Generics allow writing reusable code with type parameters. Like Java generics, they provide compile-time type safety without sacrificing flexibility. E.g., `function identity<T>(val: T): T`.

**Q: What is `never` type used for?**
> Functions that always throw or run infinitely return `never`. Also used in exhaustive union checks — if all cases are handled, the fallback type is `never`, alerting you when a new union member is added.

**Q: What are utility types?**
> Built-in generic types: `Partial<T>`, `Required<T>`, `Readonly<T>`, `Pick<T,K>`, `Omit<T,K>`, `Record<K,V>`, `Exclude<T,U>`, `Extract<T,U>`, `NonNullable<T>`, `ReturnType<T>`.

**Q: What is `strictNullChecks`?**
> When enabled, `null` and `undefined` are not assignable to other types. Forces you to handle null cases explicitly, preventing null pointer exceptions at runtime.

**Q: What is a discriminated union?**
> A union type where each member has a common literal property (discriminant) used to narrow the type. E.g., `{ kind: "circle", radius: number } | { kind: "rect", width: number }`.

**Q: What are decorators in TypeScript?**
> Decorators are special functions that modify classes, methods, properties, or parameters. Used heavily in Angular (`@Component`, `@Injectable`) and NestJS (`@Controller`, `@Get`). They implement a form of metaprogramming.

**Q: Explain `keyof` and `typeof`.**
> `keyof T` produces a union of property names of T as string literals. `typeof expr` gets the TypeScript type of an expression (not the JS runtime `typeof`).

**Q: What is the difference between `extends` and `implements` in TypeScript classes?**
> `extends` is for class inheritance (inherit implementation). `implements` is for interface conformance (just the shape, no inherited code). A class can implement multiple interfaces but extend only one class.
