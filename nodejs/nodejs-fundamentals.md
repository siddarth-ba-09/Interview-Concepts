# Node.js — Complete Fundamentals Guide

> Node.js is a JavaScript runtime built on Chrome's V8 engine that lets you run JS on the server. As a Java developer, think of it as the JVM equivalent for JavaScript.

---

## Table of Contents

1. [What is Node.js?](#1-what-is-nodejs)
2. [Event Loop (Deep Dive)](#2-event-loop-deep-dive)
3. [Modules System](#3-modules-system)
4. [npm & package.json](#4-npm--packagejson)
5. [Core Built-in Modules](#5-core-built-in-modules)
6. [File System (fs)](#6-file-system-fs)
7. [Streams & Buffers](#7-streams--buffers)
8. [EventEmitter](#8-eventemitter)
9. [HTTP Module](#9-http-module)
10. [Path Module](#10-path-module)
11. [OS Module](#11-os-module)
12. [Process & Environment](#12-process--environment)
13. [Child Processes](#13-child-processes)
14. [Clustering & Worker Threads](#14-clustering--worker-threads)
15. [Error Handling Patterns](#15-error-handling-patterns)
16. [Async Patterns in Node.js](#16-async-patterns-in-nodejs)
17. [Security Considerations](#17-security-considerations)
18. [Miscellaneous Concepts](#18-miscellaneous-concepts)
19. [Interview Q&A](#19-interview-qa)

---

## 1. What is Node.js?

- **JavaScript runtime** built on Chrome's **V8 JavaScript engine**.
- Uses **libuv** — a C library that provides the event loop, async I/O, and thread pool.
- **Non-blocking, event-driven** architecture — ideal for I/O-heavy apps.
- **Single-threaded** for JS execution, but I/O operations run on a background thread pool.

### Node.js vs Java (Server Comparison)

| Aspect | Node.js | Java (Spring Boot) |
|--------|---------|-------------------|
| Concurrency | Event-driven, async I/O | Thread-per-request (or reactive) |
| Performance | Excellent for I/O-heavy | Excellent for CPU-heavy |
| Type system | Dynamic (TS adds static) | Static |
| Package manager | npm/yarn/pnpm | Maven/Gradle |
| Module system | CommonJS / ESM | — |
| Default HTTP server | Single thread (non-blocking) | Thread pool (Tomcat) |

### When to use Node.js
- REST APIs, real-time apps (chat, live feeds)
- Microservices
- API gateways, BFF (Backend for Frontend)
- Streaming data
- CLI tools

### Not ideal for
- CPU-intensive computation (video encoding, ML)
- Long blocking operations

---

## 2. Event Loop (Deep Dive)

Node.js event loop has **phases**, unlike the browser which just has micro/macro queues.

```
   ┌──────────────────────────────────────┐
   │           timers                     │  setTimeout, setInterval callbacks
   ├──────────────────────────────────────┤
   │       pending callbacks              │  I/O errors from prev iteration
   ├──────────────────────────────────────┤
   │           idle, prepare             │  internal only
   ├──────────────────────────────────────┤
   │              poll                    │  retrieve new I/O events ← blocks here
   ├──────────────────────────────────────┤
   │             check                    │  setImmediate callbacks
   ├──────────────────────────────────────┤
   │        close callbacks               │  socket.on("close", ...)
   └──────────────────────────────────────┘
```

**Between each phase:** `process.nextTick()` queue + **Microtask queue** (Promises) run first.

### Execution Priority (highest to lowest)
1. **Synchronous code** (call stack)
2. **`process.nextTick()`** queue
3. **Promise microtasks** (.then, .catch, queueMicrotask)
4. **setImmediate** (check phase)
5. **setTimeout/setInterval** (timers phase)
6. **I/O callbacks**

```js
console.log("1 - sync");

setTimeout(() => console.log("5 - setTimeout"), 0);

setImmediate(() => console.log("4 - setImmediate"));

Promise.resolve().then(() => console.log("3 - Promise"));

process.nextTick(() => console.log("2 - nextTick"));

console.log("1.5 - sync");

// Output: 1-sync, 1.5-sync, 2-nextTick, 3-Promise, 4-setImmediate, 5-setTimeout
```

> **Key insight:** `process.nextTick` runs before Promises, which runs before `setImmediate`, which runs before `setTimeout`.

### setImmediate vs setTimeout(fn, 0)
- Both run "after current operation completes".
- Inside an I/O callback, `setImmediate` always runs before `setTimeout`.
- Outside I/O, order is non-deterministic (OS-dependent).

---

## 3. Modules System

### CommonJS (default in Node.js)
```js
// math.js — exporting
const add = (a, b) => a + b;
const subtract = (a, b) => a - b;
const PI = 3.14;

module.exports = { add, subtract, PI }; // object export
// OR
exports.multiply = (a, b) => a * b;     // property-by-property export
// NOTE: never mix module.exports and exports.something together

// app.js — importing
const { add, PI } = require("./math");
const math = require("./math");        // entire module
const express = require("express");    // node_modules
```

### ES Modules (ESM)
```js
// package.json: "type": "module" OR use .mjs extension

// math.mjs — exporting
export const add = (a, b) => a + b;
export default function main() {}

// app.mjs — importing
import main, { add } from "./math.mjs";
import * as math from "./math.mjs";
import("./math.mjs").then(m => { }); // dynamic import

// ESM doesn't have __dirname/__filename
import { fileURLToPath } from "url";
import { dirname } from "path";
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

### Module Resolution Algorithm
```
require("express")
  → ./node_modules/express
  → ../node_modules/express
  → ../../node_modules/express
  → ... (up to root)

require("./utils")
  → ./utils.js
  → ./utils.json
  → ./utils/index.js
```

### Module Caching
```js
// Modules are cached after first require — subsequent requires return same object
const a = require("./module"); // loads module
const b = require("./module"); // returns cached version
a === b; // true

// Force reload (rare need)
delete require.cache[require.resolve("./module")];
```

---

## 4. npm & package.json

### package.json
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "My Node.js app",
  "main": "index.js",           // entry point for require()
  "type": "module",             // use ESM (omit for CommonJS)
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "build": "tsc"
  },
  "dependencies": {             // production deps
    "express": "^4.18.2"
  },
  "devDependencies": {          // dev-only deps
    "nodemon": "^3.0.0",
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### Version Ranges
- `^4.18.2` — compatible with 4.x.x (minor/patch updates ok)
- `~4.18.2` — approximately equal (patch updates only)
- `4.18.2` — exact version
- `>=4.0.0` — greater than or equal
- `*` or `""` — any version

### Essential npm Commands
```bash
npm init -y                     # create package.json
npm install express             # install & add to dependencies
npm install -D nodemon          # install as devDependency
npm install                     # install all deps from package.json
npm update                      # update packages
npm uninstall express           # remove package
npm run start                   # run script
npm list                        # list installed packages
npm audit                       # check for vulnerabilities
npm audit fix                   # fix vulnerabilities
npm cache clean --force         # clear cache
npx create-react-app my-app     # run package without installing
```

### package-lock.json vs yarn.lock
- Locks exact versions of all deps (and transitive deps) for reproducible installs.
- Always commit this file to version control.

---

## 5. Core Built-in Modules

```js
// No installation needed — built into Node.js
const fs = require("fs");
const path = require("path");
const http = require("http");
const https = require("https");
const os = require("os");
const events = require("events");
const stream = require("stream");
const crypto = require("crypto");
const url = require("url");
const querystring = require("querystring");
const util = require("util");
const child_process = require("child_process");
const cluster = require("cluster");
const buffer = require("buffer");
const timers = require("timers");
```

---

## 6. File System (fs)

```js
const fs = require("fs");
const fsPromises = require("fs/promises"); // Promise-based API

// ─── Synchronous (blocks event loop — avoid in production) ───
const content = fs.readFileSync("file.txt", "utf8");
fs.writeFileSync("file.txt", "content");

// ─── Callback-based (old style) ───
fs.readFile("file.txt", "utf8", (err, data) => {
  if (err) throw err;
  console.log(data);
});

// ─── Promise-based (modern — preferred) ───
async function readFile() {
  const content = await fsPromises.readFile("file.txt", "utf8");
  await fsPromises.writeFile("output.txt", content);
  await fsPromises.appendFile("log.txt", "new line\n");
  await fsPromises.unlink("temp.txt");     // delete file
  await fsPromises.rename("old.txt", "new.txt");
  await fsPromises.mkdir("./new-dir", { recursive: true });
  await fsPromises.rmdir("./old-dir", { recursive: true });
  const files = await fsPromises.readdir("./src"); // list directory
  const stats = await fsPromises.stat("file.txt"); // file metadata
  stats.isFile();        // true
  stats.isDirectory();   // false
  stats.size;            // bytes
}

// ─── Watch for changes ───
fs.watch("file.txt", (eventType, filename) => {
  console.log(`${filename} changed: ${eventType}`);
});
```

---

## 7. Streams & Buffers

### Buffer
```js
// Fixed-length raw binary data
const buf = Buffer.from("Hello", "utf8");
buf.toString("utf8");     // "Hello"
buf.toString("hex");      // "48656c6c6f"
buf.toString("base64");   // "SGVsbG8="
buf.length;               // 5

const buf2 = Buffer.alloc(10);        // zero-filled buffer of 10 bytes
const buf3 = Buffer.allocUnsafe(10);  // faster but may contain old data
Buffer.concat([buf, buf2]);           // merge buffers
```

### Streams (key concept for large data)
Streams process data **in chunks** instead of loading everything into memory.

```
Readable Stream → Transform Stream → Writable Stream
```

```js
const fs = require("fs");
const zlib = require("zlib");

// ─── Readable Stream ───
const readable = fs.createReadStream("large-file.txt", { encoding: "utf8", highWaterMark: 64 * 1024 });

readable.on("data", (chunk) => console.log("chunk:", chunk.length));
readable.on("end", () => console.log("Done"));
readable.on("error", (err) => console.error(err));

// ─── Writable Stream ───
const writable = fs.createWriteStream("output.txt");
writable.write("Hello ");
writable.write("World");
writable.end(); // signal end of writing
writable.on("finish", () => console.log("Written"));

// ─── Pipe (connects readable to writable) ───
fs.createReadStream("input.txt")
  .pipe(zlib.createGzip())             // transform stream (compress)
  .pipe(fs.createWriteStream("input.txt.gz"));

// ─── Pipeline (better error handling) ───
const { pipeline } = require("stream/promises");
await pipeline(
  fs.createReadStream("input.txt"),
  zlib.createGzip(),
  fs.createWriteStream("output.gz")
);

// ─── Custom Readable ───
const { Readable } = require("stream");
const customReadable = new Readable({
  read(size) {
    this.push("data chunk");
    this.push(null); // signal end
  }
});
```

### Stream Types

| Type | Description | Example |
|------|-------------|---------|
| Readable | Source of data | `fs.createReadStream`, `http.IncomingMessage` |
| Writable | Destination for data | `fs.createWriteStream`, `http.ServerResponse` |
| Duplex | Both readable & writable | `net.Socket`, `TCP socket` |
| Transform | Modify data as it passes | `zlib.createGzip()`, crypto streams |

---

## 8. EventEmitter

Node.js is built on an event-driven architecture. Most core modules (HTTP, streams, fs) extend `EventEmitter`.

```js
const EventEmitter = require("events");

class OrderService extends EventEmitter {
  placeOrder(order) {
    console.log("Processing order:", order.id);
    this.emit("order:placed", order);          // emit event with data
    this.emit("order:invoice", order, 0.1);    // multiple args
  }

  cancelOrder(orderId) {
    this.emit("order:cancelled", { id: orderId });
  }
}

const service = new OrderService();

// Register listeners
service.on("order:placed", (order) => {
  console.log("Send confirmation email for:", order.id);
});

service.on("order:placed", (order) => {
  console.log("Update inventory for:", order.id);
});

service.once("order:placed", (order) => {
  console.log("One-time logic for first order");
}); // fires only once

service.on("error", (err) => {    // always handle "error" event
  console.error("OrderService error:", err);
});

// Remove listener
const handler = (order) => console.log(order);
service.on("order:placed", handler);
service.off("order:placed", handler);  // or service.removeListener(...)

// Max listeners (default 10, warn if exceeded)
service.setMaxListeners(20);

service.placeOrder({ id: "ORD-001" });

// List events
service.eventNames();            // ["order:placed", "error"]
service.listenerCount("order:placed"); // 2
```

---

## 9. HTTP Module

```js
const http = require("http");
const url = require("url");

// ─── Basic HTTP Server ───
const server = http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url, true); // true = parse query string
  const pathname = parsedUrl.pathname;
  const query = parsedUrl.query;
  const method = req.method;

  // Read request body
  let body = "";
  req.on("data", chunk => { body += chunk.toString(); });
  req.on("end", () => {
    const data = body ? JSON.parse(body) : null;

    if (method === "GET" && pathname === "/users") {
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ users: [] }));
    } else if (method === "POST" && pathname === "/users") {
      res.writeHead(201, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ created: data }));
    } else {
      res.writeHead(404, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ error: "Not Found" }));
    }
  });
});

server.listen(3000, () => console.log("Server running on port 3000"));

// ─── HTTP Client (making requests) ───
const options = {
  hostname: "jsonplaceholder.typicode.com",
  path: "/posts/1",
  method: "GET"
};

const req = http.request(options, (res) => {
  let data = "";
  res.on("data", chunk => data += chunk);
  res.on("end", () => console.log(JSON.parse(data)));
});
req.on("error", console.error);
req.end();
```

> In practice, always use **Express** or **Fastify** instead of raw http module.

---

## 10. Path Module

```js
const path = require("path");

path.join("/users", "alice", "docs", "file.txt"); // /users/alice/docs/file.txt
path.join("a", "..", "b"); // b  (resolves ..)

path.resolve("src", "index.js");     // /current/working/dir/src/index.js (absolute)
path.resolve("/etc", "nginx.conf");  // /etc/nginx.conf

path.basename("/path/to/file.txt");  // "file.txt"
path.basename("/path/to/file.txt", ".txt"); // "file"
path.dirname("/path/to/file.txt");   // "/path/to"
path.extname("file.txt");            // ".txt"

path.parse("/path/to/file.txt");
// { root: "/", dir: "/path/to", base: "file.txt", ext: ".txt", name: "file" }

path.format({ dir: "/path/to", name: "file", ext: ".txt" }); // "/path/to/file.txt"

path.sep;     // "/" on Unix, "\" on Windows
path.delimiter; // ":" on Unix, ";" on Windows
path.isAbsolute("/path"); // true
path.isAbsolute("path");  // false

// Relative path between two paths
path.relative("/data/orandea/test/aaa", "/data/orandea/impl/bbb"); // "../../impl/bbb"
```

---

## 11. OS Module

```js
const os = require("os");

os.platform();    // "darwin" | "linux" | "win32"
os.arch();        // "x64" | "arm64"
os.cpus();        // array of CPU info objects
os.cpus().length; // number of CPU cores
os.totalmem();    // total memory in bytes
os.freemem();     // free memory in bytes
os.hostname();    // computer hostname
os.homedir();     // /home/alice or C:\Users\Alice
os.tmpdir();      // /tmp or C:\Temp
os.networkInterfaces(); // network interface info
os.uptime();      // system uptime in seconds
os.EOL;           // "\n" on Unix, "\r\n" on Windows
os.type();        // "Linux" | "Darwin" | "Windows_NT"
```

---

## 12. Process & Environment

```js
// process is a global — no require needed

process.env.NODE_ENV;          // "development" | "production" | "test"
process.env.PORT;              // "3000" (always a string!)
const port = parseInt(process.env.PORT ?? "3000", 10);

process.argv;                  // ["node", "script.js", "--flag", "value"]
process.argv.slice(2);         // actual arguments after node and script

process.cwd();                 // current working directory
process.chdir("/new/path");    // change working directory

process.pid;                   // process ID
process.version;               // "v20.0.0"
process.platform;              // "linux" | "darwin" | "win32"
process.memoryUsage();         // { rss, heapTotal, heapUsed, external }
process.cpuUsage();            // { user, system }

process.exit(0);               // exit with success code
process.exit(1);               // exit with error code

process.stdout.write("Hello\n"); // console.log equivalent
process.stderr.write("Error\n");
process.stdin.on("data", (data) => console.log("Input:", data.toString().trim()));

// Signals
process.on("SIGTERM", () => {
  console.log("Gracefully shutting down...");
  server.close(() => process.exit(0));
});

process.on("SIGINT", () => {    // Ctrl+C
  process.exit(0);
});

// Uncaught exceptions
process.on("uncaughtException", (err) => {
  console.error("Uncaught:", err);
  process.exit(1); // always exit after uncaughtException
});

process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled Rejection:", reason);
});
```

### dotenv (Loading .env files)
```js
// npm install dotenv
require("dotenv").config(); // load .env into process.env

// .env file (never commit to git!)
// DB_HOST=localhost
// DB_PORT=5432
// JWT_SECRET=my-secret-key
```

---

## 13. Child Processes

Run shell commands or spawn other processes from Node.js.

```js
const { exec, execFile, spawn, fork } = require("child_process");

// ─── exec — runs command in shell, buffers output ───
exec("ls -la", (err, stdout, stderr) => {
  if (err) { console.error(err); return; }
  console.log(stdout);
});

// ─── execFile — safer, no shell, runs file directly ───
execFile("node", ["--version"], (err, stdout) => console.log(stdout));

// ─── spawn — streaming, better for large output ───
const child = spawn("ping", ["-c", "4", "google.com"]);
child.stdout.on("data", (data) => process.stdout.write(data));
child.stderr.on("data", (data) => process.stderr.write(data));
child.on("close", (code) => console.log("Exited:", code));

// ─── fork — spawns a new Node.js process, has IPC channel ───
const worker = fork("./worker.js");
worker.send({ task: "process-data", payload: [1, 2, 3] });
worker.on("message", (result) => console.log("Worker result:", result));
worker.on("exit", (code) => console.log("Worker exited:", code));

// worker.js
process.on("message", (msg) => {
  const result = msg.payload.reduce((a, b) => a + b, 0);
  process.send({ result });
});
```

---

## 14. Clustering & Worker Threads

### Clustering (multiple processes, one per CPU core)
```js
const cluster = require("cluster");
const http = require("http");
const os = require("os");

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  console.log(`Primary process ${process.pid} forking ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker, code) => {
    console.log(`Worker ${worker.process.pid} died. Restarting...`);
    cluster.fork(); // restart crashed workers
  });
} else {
  // Each worker runs its own HTTP server
  http.createServer((req, res) => {
    res.end(`Handled by worker ${process.pid}`);
  }).listen(3000);
  console.log(`Worker ${process.pid} started`);
}
```

### Worker Threads (CPU-intensive tasks)
```js
const { Worker, isMainThread, parentPort, workerData } = require("worker_threads");

if (isMainThread) {
  const worker = new Worker(__filename, { workerData: { num: 100000000 } });
  worker.on("message", (result) => console.log("Result:", result));
  worker.on("error", console.error);
  worker.on("exit", (code) => console.log("Exit:", code));
} else {
  // CPU-intensive task — won't block main thread
  let sum = 0;
  for (let i = 0; i < workerData.num; i++) sum += i;
  parentPort.postMessage(sum);
}
```

**Cluster vs Worker Threads:**
- **Cluster:** Multiple Node.js processes, each with own event loop and memory. Best for I/O scalability.
- **Worker Threads:** Multiple threads in same process, shared memory via SharedArrayBuffer. Best for CPU-intensive tasks.

---

## 15. Error Handling Patterns

```js
// ─── Callback error-first convention ───
function readConfig(path, callback) {
  fs.readFile(path, "utf8", (err, data) => {
    if (err) return callback(err, null);  // error first
    callback(null, JSON.parse(data));      // null for error on success
  });
}

// ─── Custom Error classes ───
class AppError extends Error {
  constructor(message, statusCode = 500, isOperational = true) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404);
  }
}

class ValidationError extends AppError {
  constructor(message) {
    super(message, 400);
  }
}

// ─── Async error handling ───
async function getUser(id) {
  try {
    const user = await User.findById(id);
    if (!user) throw new NotFoundError("User");
    return user;
  } catch (err) {
    if (err instanceof AppError) throw err;
    throw new AppError("Database error");
  }
}

// ─── Domain-level error handling (for EventEmitter errors) ───
const domain = require("domain");
const d = domain.create();
d.on("error", (err) => console.error("Domain error:", err));
d.run(() => { /* code inside domain */ });
```

---

## 16. Async Patterns in Node.js

### Promisifying Callback-based APIs
```js
const util = require("util");
const fs = require("fs");

// util.promisify converts callback-style to Promise
const readFile = util.promisify(fs.readFile);
const writeFile = util.promisify(fs.writeFile);

async function copyFile(src, dest) {
  const content = await readFile(src, "utf8");
  await writeFile(dest, content);
}

// fs.promises (modern — no need to promisify)
const { readFile: readFileP } = require("fs/promises");
```

### Async Iteration
```js
async function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) {
    await new Promise(resolve => setTimeout(resolve, 100));
    yield i;
  }
}

async function main() {
  for await (const num of generateSequence(1, 5)) {
    console.log(num); // 1, 2, 3, 4, 5 (with delay)
  }
}

// Reading file line by line
const readline = require("readline");
const rl = readline.createInterface({ input: fs.createReadStream("file.txt") });
for await (const line of rl) {
  console.log(line);
}
```

### Connection Pooling (Database)
```js
// Always use connection pools for databases
const { Pool } = require("pg");
const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,                // max connections in pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

async function query(sql, params) {
  const client = await pool.connect();
  try {
    return await client.query(sql, params);
  } finally {
    client.release(); // always release back to pool
  }
}
```

---

## 17. Security Considerations

```js
// ─── Never trust user input ───
const path = require("path");
const SAFE_DIR = "/data/uploads";

function safePath(userInput) {
  const resolved = path.resolve(SAFE_DIR, userInput);
  if (!resolved.startsWith(SAFE_DIR)) {
    throw new Error("Path traversal detected");
  }
  return resolved;
}

// ─── Use crypto for secure operations ───
const crypto = require("crypto");

// Hash passwords with bcrypt (not crypto!)
// npm install bcrypt
const bcrypt = require("bcrypt");
const hash = await bcrypt.hash("password", 12);
const match = await bcrypt.compare("password", hash);

// Secure random tokens
const token = crypto.randomBytes(32).toString("hex");
const uuid = crypto.randomUUID();

// ─── Environment variables for secrets ───
// NEVER hardcode secrets in code
const jwtSecret = process.env.JWT_SECRET;

// ─── Limit request sizes ───
// Express: app.use(express.json({ limit: "1mb" }));

// ─── Set security headers ───
// Use helmet middleware: npm install helmet
const helmet = require("helmet");
app.use(helmet());

// ─── Rate limiting ───
// npm install express-rate-limit
```

---

## 18. Miscellaneous Concepts

### util Module
```js
const util = require("util");
util.promisify(fn);          // callback → Promise
util.callbackify(asyncFn);   // Promise → callback
util.inspect(obj, { depth: null, colors: true }); // deep object logging
util.format("%s has %d users", "App", 42); // "App has 42 users"
util.isDeepStrictEqual(a, b); // deep comparison
util.deprecate(fn, "Use newFn instead"); // mark function deprecated
```

### crypto Module
```js
const crypto = require("crypto");

// Hashing
crypto.createHash("sha256").update("data").digest("hex");
crypto.createHash("md5").update("data").digest("base64");

// HMAC
crypto.createHmac("sha256", "secret").update("data").digest("hex");

// Encryption/Decryption
const key = crypto.scryptSync("password", "salt", 32);
const iv = crypto.randomBytes(16);
const cipher = crypto.createCipheriv("aes-256-cbc", key, iv);
let encrypted = cipher.update("plaintext", "utf8", "hex");
encrypted += cipher.final("hex");

const decipher = crypto.createDecipheriv("aes-256-cbc", key, iv);
let decrypted = decipher.update(encrypted, "hex", "utf8");
decrypted += decipher.final("utf8");
```

### URL Module
```js
const { URL, URLSearchParams } = require("url");

const url = new URL("https://example.com:8080/path?name=alice&page=1#section");
url.protocol; // "https:"
url.hostname; // "example.com"
url.port;     // "8080"
url.pathname; // "/path"
url.search;   // "?name=alice&page=1"
url.hash;     // "#section"
url.searchParams.get("name");  // "alice"
url.searchParams.append("sort", "asc");
url.searchParams.toString();   // "name=alice&page=1&sort=asc"
```

### Timer Functions
```js
setTimeout(fn, 1000);         // run after 1 second
setInterval(fn, 1000);        // run every 1 second
setImmediate(fn);             // run after current event loop iteration
process.nextTick(fn);         // run before next event loop iteration

clearTimeout(id);
clearInterval(id);
clearImmediate(id);

// Promisified timers
const { setTimeout: sleep } = require("timers/promises");
await sleep(1000); // await to pause async function for 1s
```

---

## 19. Interview Q&A

**Q: What is Node.js and why is it non-blocking?**
> Node.js is a JS runtime built on V8. It uses an event loop + libuv thread pool for I/O. When an I/O operation starts, Node registers a callback and continues executing other code. When I/O completes, the callback is queued for the event loop — thus non-blocking.

**Q: What is the event loop in Node.js?**
> The event loop is the mechanism that allows Node to perform non-blocking I/O. It has phases: timers → pending callbacks → idle/prepare → poll → check → close callbacks. Between each phase, microtasks (nextTick, Promises) run.

**Q: What is the difference between `process.nextTick` and `setImmediate`?**
> `process.nextTick` runs before the next event loop iteration starts (after current sync code, before I/O events). `setImmediate` runs in the "check" phase of the event loop, after I/O events.

**Q: What is the difference between `require` (CommonJS) and `import` (ESM)?**
> CommonJS is synchronous, has `require/module.exports`, widely supported. ESM is asynchronous, uses `import/export`, supports tree shaking, and requires `"type": "module"` or `.mjs` extension. ESM is the modern standard.

**Q: How does Node.js handle concurrent requests if it's single-threaded?**
> Node.js is single-threaded for JS execution but uses libuv's thread pool for I/O (file system, DNS, crypto). When handling HTTP requests, Node starts I/O, registers callbacks, then handles other requests. The event loop picks up the callbacks when I/O completes. For CPU-intensive work, use Worker Threads.

**Q: What are Streams and why use them?**
> Streams process data in chunks instead of loading the entire data into memory. Crucial for large files, HTTP responses, or real-time data. Types: Readable, Writable, Duplex, Transform. Use `pipe()` or `pipeline()` to chain them.

**Q: What is EventEmitter?**
> EventEmitter is Node's implementation of the Observer pattern. Objects that extend EventEmitter can emit named events and register listeners. Core modules like streams, HTTP server, and child processes are EventEmitters.

**Q: How do you handle uncaught exceptions in Node.js?**
> Use `process.on("uncaughtException", handler)` for sync errors and `process.on("unhandledRejection", handler)` for async Promise rejections. After `uncaughtException`, always exit the process (state may be corrupted).

**Q: What is clustering in Node.js?**
> The cluster module forks multiple Node.js processes (workers), one per CPU core. The primary process distributes incoming connections to workers. This allows taking advantage of multi-core CPUs for I/O-intensive apps.
