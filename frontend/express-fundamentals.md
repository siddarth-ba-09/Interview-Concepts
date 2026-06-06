# Express.js — Complete Fundamentals Guide

> Express is a minimal, unopinionated web framework for Node.js. Think of it as Spring MVC but without the heavy annotations — you wire everything up manually.

---

## Table of Contents

1. [What is Express?](#1-what-is-express)
2. [Basic Server Setup](#2-basic-server-setup)
3. [Routing](#3-routing)
4. [Middleware](#4-middleware)
5. [Request Object](#5-request-object)
6. [Response Object](#6-response-object)
7. [Route Parameters, Query & Body](#7-route-parameters-query--body)
8. [Router (Modular Routes)](#8-router-modular-routes)
9. [Error Handling](#9-error-handling)
10. [Static Files](#10-static-files)
11. [Template Engines (Brief)](#11-template-engines-brief)
12. [CORS](#12-cors)
13. [Security Middleware](#13-security-middleware)
14. [Authentication Patterns](#14-authentication-patterns)
15. [Database Integration Patterns](#15-database-integration-patterns)
16. [File Uploads](#16-file-uploads)
17. [REST API Best Practices](#17-rest-api-best-practices)
18. [Project Structure](#18-project-structure)
19. [Miscellaneous Concepts](#19-miscellaneous-concepts)
20. [Interview Q&A](#20-interview-qa)

---

## 1. What is Express?

- **Minimal Node.js web framework** — routing, middleware, request/response helpers.
- Unopinionated — you choose your structure, ORM, auth strategy, etc.
- Analogy: Express ≈ bare-bones Spring MVC without Spring's IoC container.
- Most widely used Node.js web framework.

```bash
npm install express
npm install -D @types/express  # TypeScript type definitions
```

---

## 2. Basic Server Setup

```js
const express = require("express");
const app = express();

// Built-in middleware
app.use(express.json());                        // parse JSON request bodies
app.use(express.urlencoded({ extended: true })); // parse form data

const PORT = process.env.PORT || 3000;

const server = app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});

// Graceful shutdown
process.on("SIGTERM", () => {
  server.close(() => {
    console.log("Server closed");
    process.exit(0);
  });
});
```

### With TypeScript
```ts
import express, { Application, Request, Response, NextFunction } from "express";

const app: Application = express();
app.use(express.json());

app.get("/", (req: Request, res: Response) => {
  res.json({ message: "Hello World" });
});

app.listen(3000);
```

---

## 3. Routing

### Basic Routes
```js
// HTTP methods
app.get("/users", (req, res) => { res.json([]); });
app.post("/users", (req, res) => { res.status(201).json({}); });
app.put("/users/:id", (req, res) => { res.json({}); });
app.patch("/users/:id", (req, res) => { res.json({}); });
app.delete("/users/:id", (req, res) => { res.status(204).send(); });

// Handle all HTTP methods
app.all("/secret", (req, res) => { res.send("Forbidden"); });

// Route patterns
app.get("/users/:id", handler);          // named param
app.get("/files/*", handler);            // wildcard
app.get(/\/users\/\d+/, handler);        // regex
app.get(["/about", "/info"], handler);   // multiple paths
```

### Chaining Route Handlers
```js
app.route("/users")
  .get((req, res) => res.json([]))
  .post((req, res) => res.status(201).json({}));

app.route("/users/:id")
  .get((req, res) => res.json({}))
  .put((req, res) => res.json({}))
  .delete((req, res) => res.status(204).send());
```

### Multiple Handlers (Middleware chain)
```js
app.get("/protected",
  authenticate,           // middleware 1
  authorize("admin"),     // middleware 2 (factory)
  (req, res) => res.json({ data: "secret" }) // final handler
);
```

---

## 4. Middleware

Middleware functions have access to `req`, `res`, and `next`. They can:
- Execute any code
- Modify req/res
- End the request-response cycle
- Call `next()` to pass to next middleware

```
Request → Middleware 1 → Middleware 2 → Route Handler → Response
                             ↓ (if error)
                       Error Handler
```

### Types of Middleware

```js
// ─── Application-level middleware ───
app.use((req, res, next) => {           // applies to ALL routes
  console.log(`${req.method} ${req.path} - ${new Date().toISOString()}`);
  next(); // MUST call next() or send a response
});

app.use("/api", (req, res, next) => {   // applies only to /api/* routes
  console.log("API request");
  next();
});

// ─── Route-level middleware ───
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ error: "Unauthorized" });
  req.user = verifyToken(token); // attach to req for downstream use
  next();
};

app.get("/dashboard", authenticate, (req, res) => {
  res.json({ user: req.user });
});

// ─── Middleware factory (higher-order function) ───
const authorize = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user?.role)) {
    return res.status(403).json({ error: "Forbidden" });
  }
  next();
};

app.delete("/users/:id", authenticate, authorize("admin"), deleteUser);
```

### Built-in Middleware
```js
app.use(express.json({ limit: "10kb" }));  // parse JSON body
app.use(express.urlencoded({ extended: true })); // parse URL-encoded form data
app.use(express.static("public"));         // serve static files
app.use(express.raw());                    // parse raw buffer body
app.use(express.text());                   // parse text/plain body
```

### Third-party Middleware
```bash
npm install morgan helmet cors compression express-rate-limit
```

```js
const morgan = require("morgan");
const helmet = require("helmet");
const cors = require("cors");
const compression = require("compression");
const rateLimit = require("express-rate-limit");

app.use(morgan("combined")); // HTTP request logger
app.use(helmet());           // security headers
app.use(cors());             // CORS headers
app.use(compression());      // gzip compression

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                  // max 100 requests per window
  message: "Too many requests, please try again later"
});
app.use("/api", limiter);
```

---

## 5. Request Object

```js
app.get("/example", (req, res) => {
  // URL parameters
  req.params;            // { id: "123" } from /users/:id
  req.params.id;

  // Query string
  req.query;             // { page: "1", limit: "10" } from ?page=1&limit=10
  req.query.page;

  // Request body (requires express.json() middleware)
  req.body;              // parsed JSON or form data
  req.body.name;

  // Headers
  req.headers;
  req.headers["content-type"];
  req.get("Content-Type");  // same, case-insensitive

  // Metadata
  req.method;            // "GET", "POST", etc.
  req.path;              // "/example"
  req.url;               // "/example?page=1"
  req.originalUrl;       // full URL including mount path
  req.hostname;          // "localhost" or "example.com"
  req.ip;                // client IP address
  req.protocol;          // "http" or "https"
  req.secure;            // true if HTTPS
  req.xhr;               // true if XMLHttpRequest

  // Cookies (requires cookie-parser middleware)
  req.cookies;
  req.signedCookies;

  // Custom properties added by middleware
  req.user;              // set by auth middleware
});
```

---

## 6. Response Object

```js
app.get("/example", (req, res) => {
  // Send responses
  res.send("Hello");               // any type, sets Content-Type automatically
  res.json({ key: "value" });      // sends JSON, sets Content-Type: application/json
  res.status(201).json({});        // chaining status + json
  res.sendStatus(204);             // status + default message ("No Content")
  res.sendFile("/path/to/file.pdf");

  // Status
  res.status(404);                  // set status code

  // Headers
  res.set("X-Custom-Header", "value");
  res.set({ "X-A": "1", "X-B": "2" }); // multiple headers
  res.type("json");                  // set Content-Type
  res.type("text/html");

  // Redirects
  res.redirect("/new-path");         // 302
  res.redirect(301, "/permanent");   // permanent redirect
  res.redirect("back");              // redirect to Referer header

  // Cookies
  res.cookie("session", "abc123", {
    httpOnly: true,     // not accessible via JS
    secure: true,       // HTTPS only
    sameSite: "strict", // CSRF protection
    maxAge: 3600000,    // 1 hour in ms
    path: "/"
  });
  res.clearCookie("session");

  // Download
  res.download("/path/to/file.pdf", "report.pdf");

  // Render (template engine)
  res.render("index", { title: "Home", user: req.user });

  // End without body
  res.end();
});
```

---

## 7. Route Parameters, Query & Body

```js
// ─── Route Parameters (/users/:id) ───
app.get("/users/:id", (req, res) => {
  const { id } = req.params;
  // Always validate — params are strings
  const userId = parseInt(id, 10);
  if (isNaN(userId)) return res.status(400).json({ error: "Invalid ID" });
  res.json({ userId });
});

// Multiple params
app.get("/orgs/:orgId/repos/:repoId", (req, res) => {
  const { orgId, repoId } = req.params;
  res.json({ orgId, repoId });
});

// ─── Query String (?page=1&limit=10&sort=name) ───
app.get("/users", (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = Math.min(parseInt(req.query.limit) || 10, 100); // cap at 100
  const sort = req.query.sort || "createdAt";
  const order = req.query.order === "desc" ? "desc" : "asc";

  res.json({ page, limit, sort, order });
});

// ─── Request Body ───
app.post("/users", (req, res) => {
  const { name, email, age } = req.body;

  // Validate
  if (!name || !email) {
    return res.status(400).json({ error: "name and email are required" });
  }

  res.status(201).json({ id: 1, name, email, age });
});
```

---

## 8. Router (Modular Routes)

```js
// ─── routes/users.js ───
const express = require("express");
const router = express.Router();

router.use((req, res, next) => {  // middleware only for this router
  console.log("User router middleware");
  next();
});

router.get("/", getUsers);
router.post("/", createUser);
router.get("/:id", getUserById);
router.put("/:id", updateUser);
router.delete("/:id", deleteUser);

module.exports = router;

// ─── app.js ───
const userRouter = require("./routes/users");
const productRouter = require("./routes/products");

app.use("/api/v1/users", userRouter);
app.use("/api/v1/products", productRouter);
// GET /api/v1/users → router.get("/")
// GET /api/v1/users/123 → router.get("/:id")
```

### Controller Pattern
```js
// ─── controllers/userController.js ───
const UserService = require("../services/userService");

exports.getUsers = async (req, res, next) => {
  try {
    const users = await UserService.findAll(req.query);
    res.json({ success: true, data: users });
  } catch (err) {
    next(err); // pass to error handler
  }
};

exports.createUser = async (req, res, next) => {
  try {
    const user = await UserService.create(req.body);
    res.status(201).json({ success: true, data: user });
  } catch (err) {
    next(err);
  }
};
```

---

## 9. Error Handling

### Synchronous Errors
```js
app.get("/test", (req, res, next) => {
  try {
    const result = riskySync();
    res.json(result);
  } catch (err) {
    next(err); // pass to error handler
  }
});
```

### Async Errors
```js
// Express 5 (not yet stable) handles async errors automatically.
// In Express 4, you must catch and pass to next():
app.get("/users", async (req, res, next) => {
  try {
    const users = await User.findAll();
    res.json(users);
  } catch (err) {
    next(err); // IMPORTANT: must call next(err)
  }
});

// Utility wrapper to avoid try-catch everywhere
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get("/users", asyncHandler(async (req, res) => {
  const users = await User.findAll();
  res.json(users);
}));
```

### Error Handling Middleware (4 parameters — must be last!)
```js
// Custom error class
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

// 404 handler (after all routes)
app.use((req, res, next) => {
  next(new AppError(`Route ${req.originalUrl} not found`, 404));
});

// Global error handler (MUST have 4 params: err, req, res, next)
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.isOperational ? err.message : "Internal Server Error";

  if (process.env.NODE_ENV === "development") {
    return res.status(statusCode).json({
      success: false,
      message,
      error: err.message,
      stack: err.stack
    });
  }

  res.status(statusCode).json({ success: false, message });
});
```

---

## 10. Static Files

```js
// Serve everything in "public" folder
app.use(express.static("public"));
// GET /image.png → looks in public/image.png
// GET /css/style.css → looks in public/css/style.css

// With virtual prefix
app.use("/static", express.static("public"));
// GET /static/image.png → looks in public/image.png

// Multiple static directories
app.use(express.static("public"));
app.use(express.static("uploads"));

// Options
app.use(express.static("public", {
  maxAge: "1d",         // cache control
  etag: true,           // ETag header
  index: "index.html",  // default file for directories
  dotfiles: "ignore"    // don't serve .dotfiles
}));
```

---

## 11. Template Engines (Brief)

Express supports EJS, Pug, Handlebars. In full-stack Java dev context, you'll likely use React/Angular instead.

```js
// EJS
app.set("view engine", "ejs");
app.set("views", "./views");

app.get("/", (req, res) => {
  res.render("index", { title: "Home", users: [] });
});

// views/index.ejs
// <h1><%= title %></h1>
// <% users.forEach(u => { %> <p><%= u.name %></p> <% }) %>
```

---

## 12. CORS

Cross-Origin Resource Sharing — browser security that restricts requests from different origins.

```js
const cors = require("cors");

// Allow all origins (development only!)
app.use(cors());

// Specific configuration
app.use(cors({
  origin: ["http://localhost:4200", "https://myapp.com"], // allowed origins
  methods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
  allowedHeaders: ["Content-Type", "Authorization"],
  credentials: true,       // allow cookies/auth headers
  maxAge: 86400            // preflight cache (seconds)
}));

// Dynamic origin
app.use(cors({
  origin: (origin, callback) => {
    const whitelist = ["http://localhost:3000", "http://localhost:4200"];
    if (!origin || whitelist.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS"));
    }
  }
}));

// Enable CORS for specific route only
app.get("/public-data", cors(), (req, res) => {
  res.json({ data: "publicly accessible" });
});
```

---

## 13. Security Middleware

```js
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");
const mongoSanitize = require("express-mongo-sanitize");
const xss = require("xss-clean");
const hpp = require("hpp"); // HTTP Parameter Pollution

// Security Headers (sets many security-related HTTP headers)
app.use(helmet());

// Rate Limiting
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({ error: "Too many requests" });
  }
});
app.use("/api", apiLimiter);

// Login-specific stricter limit
const loginLimiter = rateLimit({
  windowMs: 10 * 60 * 1000,
  max: 5,
  message: "Too many login attempts"
});
app.post("/auth/login", loginLimiter, loginHandler);

// Data Sanitization (prevent NoSQL injection)
app.use(mongoSanitize()); // strips $ and . from req.body, query, params

// XSS Protection
app.use(xss()); // sanitize HTML in body

// HTTP Parameter Pollution
app.use(hpp());

// Body size limit
app.use(express.json({ limit: "10kb" }));
```

---

## 14. Authentication Patterns

### JWT Authentication
```js
const jwt = require("jsonwebtoken");
const bcrypt = require("bcrypt");

const JWT_SECRET = process.env.JWT_SECRET;
const JWT_EXPIRES_IN = "1h";

// Login — generate token
app.post("/auth/login", async (req, res, next) => {
  try {
    const { email, password } = req.body;
    if (!email || !password) {
      return res.status(400).json({ error: "Email and password required" });
    }

    const user = await User.findByEmail(email);
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ error: "Invalid credentials" });
    }

    const token = jwt.sign(
      { id: user.id, email: user.email, role: user.role },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRES_IN }
    );

    res.json({ token, user: { id: user.id, email: user.email } });
  } catch (err) {
    next(err);
  }
});

// Middleware — verify token
const authenticate = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith("Bearer ")) {
    return res.status(401).json({ error: "No token provided" });
  }

  const token = authHeader.split(" ")[1];
  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    if (err.name === "TokenExpiredError") {
      return res.status(401).json({ error: "Token expired" });
    }
    return res.status(401).json({ error: "Invalid token" });
  }
};

app.get("/profile", authenticate, (req, res) => {
  res.json({ user: req.user });
});
```

---

## 15. Database Integration Patterns

### With Mongoose (MongoDB)
```js
const mongoose = require("mongoose");

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log("MongoDB connected"))
  .catch(err => console.error("MongoDB error:", err));

const userSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, required: true, select: false },
  role: { type: String, enum: ["user", "admin"], default: "user" },
  createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model("User", userSchema);
```

### With PostgreSQL (pg)
```js
const { Pool } = require("pg");

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const userQueries = {
  findAll: () => pool.query("SELECT id, name, email FROM users ORDER BY created_at DESC"),
  findById: (id) => pool.query("SELECT id, name, email FROM users WHERE id = $1", [id]),
  create: (name, email, hashedPwd) =>
    pool.query("INSERT INTO users(name, email, password) VALUES($1, $2, $3) RETURNING id, name, email",
      [name, email, hashedPwd]),
  update: (id, updates) => pool.query("UPDATE users SET name=$1 WHERE id=$2 RETURNING *",
    [updates.name, id]),
  delete: (id) => pool.query("DELETE FROM users WHERE id=$1", [id])
};
```

---

## 16. File Uploads

```bash
npm install multer
```

```js
const multer = require("multer");
const path = require("path");
const crypto = require("crypto");

// Store in memory (for processing)
const memoryStorage = multer.memoryStorage();

// Store on disk
const diskStorage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, "uploads/"),
  filename: (req, file, cb) => {
    const uniqueName = `${crypto.randomBytes(16).toString("hex")}${path.extname(file.originalname)}`;
    cb(null, uniqueName);
  }
});

const upload = multer({
  storage: diskStorage,
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    const allowedTypes = ["image/jpeg", "image/png", "image/webp"];
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error("Only JPEG, PNG, WebP allowed"), false);
    }
  }
});

// Single file
app.post("/upload", upload.single("avatar"), (req, res) => {
  res.json({ filename: req.file.filename, path: req.file.path });
});

// Multiple files
app.post("/upload-multiple", upload.array("photos", 5), (req, res) => {
  res.json({ files: req.files.map(f => f.filename) });
});
```

---

## 17. REST API Best Practices

### URL Structure
```
GET    /api/v1/users          — list users
POST   /api/v1/users          — create user
GET    /api/v1/users/:id      — get user
PUT    /api/v1/users/:id      — replace user
PATCH  /api/v1/users/:id      — partial update
DELETE /api/v1/users/:id      — delete user

GET    /api/v1/users/:id/orders        — nested resource
POST   /api/v1/users/:id/orders
```

### Consistent Response Format
```json
// Success
{
  "success": true,
  "data": { "id": 1, "name": "Alice" },
  "message": "User created"
}

// Paginated list
{
  "success": true,
  "data": [...],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 10,
    "totalPages": 10
  }
}

// Error
{
  "success": false,
  "error": "Validation failed",
  "details": [
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

### Input Validation
```bash
npm install express-validator
```

```js
const { body, param, validationResult } = require("express-validator");

const validateCreateUser = [
  body("name").trim().notEmpty().withMessage("Name is required")
              .isLength({ min: 2, max: 50 }).withMessage("Name must be 2-50 chars"),
  body("email").isEmail().normalizeEmail().withMessage("Invalid email"),
  body("age").optional().isInt({ min: 1, max: 150 }).withMessage("Invalid age"),
  body("password").isLength({ min: 8 }).withMessage("Password must be at least 8 chars")
                  .matches(/\d/).withMessage("Password must contain a number")
];

const handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({
      success: false,
      error: "Validation failed",
      details: errors.array()
    });
  }
  next();
};

app.post("/users", validateCreateUser, handleValidationErrors, createUser);
```

### HTTP Status Codes Reference
```
200 OK             — successful GET, PUT, PATCH
201 Created        — successful POST
204 No Content     — successful DELETE
400 Bad Request    — validation failed, bad syntax
401 Unauthorized   — not authenticated
403 Forbidden      — authenticated but not authorized
404 Not Found      — resource doesn't exist
409 Conflict       — duplicate resource
422 Unprocessable  — semantically invalid data
429 Too Many Req   — rate limit exceeded
500 Internal Error — server error
503 Unavailable    — service down/maintenance
```

---

## 18. Project Structure

```
src/
├── app.js              # express app setup (no listen)
├── server.js           # entry point (listen here)
├── config/
│   ├── database.js
│   └── env.js
├── routes/
│   ├── index.js        # combine all routers
│   ├── users.js
│   └── products.js
├── controllers/
│   ├── userController.js
│   └── productController.js
├── services/
│   ├── userService.js  # business logic
│   └── emailService.js
├── models/
│   ├── User.js
│   └── Product.js
├── middleware/
│   ├── authenticate.js
│   ├── authorize.js
│   ├── errorHandler.js
│   ├── asyncHandler.js
│   └── validate.js
├── utils/
│   ├── ApiError.js
│   └── logger.js
└── validations/
    └── userValidation.js
```

```js
// app.js — clean setup
const express = require("express");
const cors = require("cors");
const helmet = require("helmet");
const morgan = require("morgan");
const routes = require("./routes");
const errorHandler = require("./middleware/errorHandler");

const app = express();

app.use(helmet());
app.use(cors());
app.use(morgan("dev"));
app.use(express.json({ limit: "10kb" }));

app.use("/api/v1", routes);

// 404
app.use((req, res) => res.status(404).json({ error: "Not Found" }));

// Error handler (LAST)
app.use(errorHandler);

module.exports = app;

// server.js
const app = require("./app");
const { connect } = require("./config/database");

const start = async () => {
  await connect();
  app.listen(3000, () => console.log("Server ready on port 3000"));
};
start();
```

---

## 19. Miscellaneous Concepts

### Compression
```js
const compression = require("compression");
app.use(compression({ level: 6, threshold: 1024 })); // gzip responses > 1KB
```

### Cookie Parser
```js
const cookieParser = require("cookie-parser");
app.use(cookieParser("secret-for-signing"));

app.get("/cookies", (req, res) => {
  res.cookie("pref", "dark-mode", { signed: true, httpOnly: true });
  console.log(req.cookies);       // unsigned cookies
  console.log(req.signedCookies); // signed cookies
  res.send("Cookie set");
});
```

### Session
```js
const session = require("express-session");
const RedisStore = require("connect-redis").default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    maxAge: 24 * 60 * 60 * 1000 // 1 day
  }
}));
```

### Logging with Morgan
```js
// Predefined formats: tiny, short, dev, combined, common
app.use(morgan("dev")); // development

// Custom format
app.use(morgan(":method :url :status :res[content-length] - :response-time ms"));

// Log to file in production
const fs = require("fs");
const accessLog = fs.createWriteStream("./access.log", { flags: "a" });
app.use(morgan("combined", { stream: accessLog }));
```

### Environment-based Config
```js
// config/env.js
require("dotenv").config();

module.exports = {
  nodeEnv: process.env.NODE_ENV || "development",
  port: parseInt(process.env.PORT, 10) || 3000,
  dbUri: process.env.DATABASE_URL,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || "1h",
  isDev: process.env.NODE_ENV !== "production"
};
```

---

## 20. Interview Q&A

**Q: What is middleware in Express?**
> Middleware functions are functions that have access to `req`, `res`, and `next`. They execute in the order they are added. Each middleware can process the request, modify req/res, end the cycle, or call `next()` to pass control to the next middleware.

**Q: How do you handle errors in Express?**
> Error-handling middleware takes 4 arguments `(err, req, res, next)`. You pass errors by calling `next(err)`. Place the error handler after all routes. For async routes, wrap in try-catch and call `next(err)` in the catch block.

**Q: What is the difference between `app.use` and `app.get`?**
> `app.use` matches any HTTP method and can match path prefixes. `app.get` only matches GET requests on an exact path. `app.use` is typically used for middleware; `app.get/post/put/delete` for route handlers.

**Q: How does routing work with Express Router?**
> `express.Router()` creates a modular, mountable route handler. Each router can have its own middleware and routes. Routers are mounted on the app with `app.use("/prefix", router)`.

**Q: How do you parse request bodies?**
> Use `express.json()` for `application/json` and `express.urlencoded()` for form data. These are built-in since Express 4.16+. For multipart/form-data (file uploads), use `multer`.

**Q: How do you implement authentication in Express?**
> Common approach: JWT authentication middleware. On login, sign a JWT with user info and return it. On protected routes, middleware extracts the token from the `Authorization: Bearer <token>` header, verifies it with `jwt.verify()`, and attaches the decoded user to `req.user`.

**Q: What is CORS and how do you handle it?**
> CORS is a browser security mechanism that blocks cross-origin requests. Use the `cors` npm package to configure allowed origins, methods, and headers. For APIs consumed by a different-origin frontend, CORS headers must be set on the server.

**Q: What is the difference between `app.listen` and `http.createServer`?**
> `app.listen(port)` is shorthand for `http.createServer(app).listen(port)`. Using `http.createServer(app)` explicitly gives you access to the server instance, useful for WebSocket upgrades or graceful shutdown.

**Q: How do you structure a large Express application?**
> Use the separation of concerns pattern: routes (URL mapping) → controllers (request handling) → services (business logic) → models (data access). Keep middleware in a separate folder. Use `express.Router()` to split routes by resource.
