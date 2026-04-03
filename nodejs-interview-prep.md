# Node.js Interview Preparation Guide

> Comprehensive interview-ready answers for experienced Node.js developers.

---

## Table of Contents

1. [[#1 Event Loop]]
2. [[#2 Modules — CommonJS vs ESM]]
3. [[#3 Streams]]
4. [[#4 Async Patterns]]
5. [[#5 Cluster & Worker Threads]]
6. [[#6 Buffers]]
7. [[#7 Express & Middleware]]
8. [[#8 REST API Design]]
9. [[#9 Authentication — JWT, OAuth, Sessions]]
10. [[#10 Error Handling]]
11. [[#11 Security]]
12. [[#12 Database — MongoDB & SQL]]
13. [[#13 Caching — Redis & Strategies]]
14. [[#14 Memory Leaks & Profiling]]
15. [[#15 Microservices]]
16. [[#16 Logging & Monitoring]]
17. [[#17 Docker & Deployment]]
18. [[#18 Design Patterns in Node.js]]

---

## 1 Event Loop

### Q: What is the Event Loop and how does it work in Node.js?

**Answer:**
The Event Loop is the mechanism that allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded. It offloads operations to the system kernel (via libuv) whenever possible.

**The Event Loop has 6 phases, executed in order:**

```
   ┌───────────────────────────┐
┌─>│        timers              │  ← setTimeout, setInterval callbacks
│  └──────────┬────────────────┘
│  ┌──────────┴────────────────┐
│  │     pending callbacks      │  ← I/O callbacks deferred to next iteration
│  └──────────┬────────────────┘
│  ┌──────────┴────────────────┐
│  │       idle, prepare        │  ← internal use only
│  └──────────┬────────────────┘
│  ┌──────────┴────────────────┐
│  │         poll               │  ← retrieve new I/O events; execute I/O callbacks
│  └──────────┬────────────────┘
│  ┌──────────┴────────────────┐
│  │         check              │  ← setImmediate callbacks
│  └──────────┬────────────────┘
│  ┌──────────┴────────────────┐
│  │    close callbacks         │  ← socket.on('close', ...)
│  └──────────┴────────────────┘
```

**Key Rules:**
- **Microtasks** (`process.nextTick`, `Promise.then`) run **between every phase**, not inside any phase.
- `process.nextTick` has higher priority than resolved Promises.

### Q: What is the difference between `process.nextTick()` and `setImmediate()`?

**Answer:**

| Feature | `process.nextTick()` | `setImmediate()` |
|---|---|---|
| **When it runs** | Before the event loop continues to the next phase (microtask) | During the **check** phase of the event loop |
| **Priority** | Higher — runs immediately after current operation | Lower — runs on next event loop iteration |
| **Risk** | Can starve the event loop if called recursively | Safer for recursive calls |
| **Use case** | Guarantee something runs before any I/O | Run after I/O events have been processed |

```js
// Output order:
console.log('start');
setImmediate(() => console.log('setImmediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
console.log('end');

// Result: start → end → nextTick → promise → setImmediate
```

### Q: What are microtasks and macrotasks?

**Answer:**

- **Microtasks:** `process.nextTick()`, `Promise.then/catch/finally`, `queueMicrotask()`. They are processed **after the current operation completes** and **before the event loop moves to the next phase**.
- **Macrotasks:** `setTimeout`, `setInterval`, `setImmediate`, I/O operations. They are processed **one per event loop iteration** (one from the queue, then all microtasks drain).

The entire microtask queue is drained before moving to the next macrotask.

---

## 2 Modules — CommonJS vs ESM

### Q: What is the difference between CommonJS and ES Modules?

**Answer:**

| Feature | CommonJS (`require`) | ES Modules (`import`) |
|---|---|---|
| **Loading** | Synchronous | Asynchronous |
| **Syntax** | `const x = require('x')` | `import x from 'x'` |
| **Exports** | `module.exports = ...` | `export default / export { }` |
| **Evaluation** | Runtime (dynamic) | Compile-time (static analysis possible) |
| **Top-level await** | Not supported | Supported |
| **Tree-shaking** | Not possible | Possible (static imports) |
| **File extension** | `.js` (default) or `.cjs` | `.mjs` or `.js` with `"type": "module"` in `package.json` |
| **`this` at top level** | `module.exports` | `undefined` |

### Q: What happens when you `require()` a module? Explain module caching.

**Answer:**

1. **Resolve** — Node resolves the file path (checks core modules → node_modules → file path).
2. **Load** — Reads the file from disk.
3. **Wrap** — Wraps the code in an IIFE:
   ```js
   (function(exports, require, module, __filename, __dirname) {
     // your module code
   });
   ```
4. **Execute** — Runs the wrapped function.
5. **Cache** — Stores the `module.exports` in `require.cache`.

On subsequent `require()` calls, Node returns the **cached** version. This means modules are **singletons** — they only execute once.

### Q: What are circular dependencies and how does Node handle them?

**Answer:**

When Module A requires Module B, and Module B requires Module A, Node.js returns a **partially completed** `exports` object to break the cycle. The module that is loaded second receives an incomplete version of the first module's exports.

```js
// a.js
exports.loaded = false;
const b = require('./b');
console.log('In a, b.loaded =', b.loaded); // true
exports.loaded = true;

// b.js
exports.loaded = false;
const a = require('./a');
console.log('In b, a.loaded =', a.loaded); // false (partial!)
exports.loaded = true;
```

**Best Practice:** Avoid circular dependencies. Refactor shared logic into a third module.

---

## 3 Streams

### Q: What are Streams in Node.js? What are the types?

**Answer:**

Streams are objects that let you read or write data **piece by piece** (chunks) instead of loading everything into memory at once. They are instances of `EventEmitter`.

**Four types:**
1. **Readable** — Source of data (e.g., `fs.createReadStream`, `http.IncomingMessage`).
2. **Writable** — Destination for data (e.g., `fs.createWriteStream`, `http.ServerResponse`).
3. **Duplex** — Both readable and writable (e.g., TCP socket).
4. **Transform** — Duplex stream that modifies data as it passes through (e.g., `zlib.createGzip()`).

### Q: What is backpressure and how do you handle it?

**Answer:**

Backpressure occurs when the **writable stream cannot process data as fast as the readable stream produces it**. Without handling, data accumulates in memory and can cause crashes.

**How `pipe()` handles it automatically:**
```js
readableStream.pipe(writableStream);
// pipe() pauses readable when writable's buffer is full
// and resumes it when writable drains
```

**Manual handling:**
```js
readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    readable.pause(); // Stop reading
    writable.once('drain', () => {
      readable.resume(); // Resume when writable is ready
    });
  }
});
```

### Q: What is the difference between flowing and paused mode?

**Answer:**

- **Paused mode (default):** You manually call `stream.read()` to pull data.
- **Flowing mode:** Data is read automatically and provided via events. Activated by attaching a `'data'` listener, calling `stream.resume()`, or calling `stream.pipe()`.

---

## 4 Async Patterns

### Q: Explain the evolution from callbacks to async/await.

**Answer:**

**1. Callbacks (Callback Hell):**
```js
getUser(id, (err, user) => {
  if (err) return handleError(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handleError(err);
    getOrderDetails(orders[0].id, (err, details) => {
      // deeply nested...
    });
  });
});
```

**2. Promises (Chaining):**
```js
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(err => handleError(err));
```

**3. Async/Await (Modern):**
```js
async function fetchDetails(id) {
  try {
    const user = await getUser(id);
    const orders = await getOrders(user.id);
    const details = await getOrderDetails(orders[0].id);
    return details;
  } catch (err) {
    handleError(err);
  }
}
```

### Q: Explain `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`.

**Answer:**

| Method | Behavior | Resolves when | Rejects when |
|---|---|---|---|
| `Promise.all` | Run all in parallel | **All** resolve | **Any one** rejects (fail-fast) |
| `Promise.allSettled` | Run all in parallel | **All** settle (resolve or reject) | Never rejects |
| `Promise.race` | Run all in parallel | **First one** to settle (resolve **or** reject) | First one rejects |
| `Promise.any` | Run all in parallel | **First one** to resolve | **All** reject (`AggregateError`) |

```js
// Promise.allSettled — useful for batch operations
const results = await Promise.allSettled([
  fetch('/api/users'),
  fetch('/api/orders'),
  fetch('/api/products'),
]);

results.forEach(result => {
  if (result.status === 'fulfilled') {
    console.log(result.value);
  } else {
    console.error(result.reason);
  }
});
```

---

## 5 Cluster & Worker Threads

### Q: How does the Cluster module work? When would you use it?

**Answer:**

The Cluster module allows you to create **child processes (workers)** that share the same server port. Each worker is a separate Node.js process with its own event loop and memory.

```js
const cluster = require('cluster');
const os = require('os');
const http = require('http');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork(); // Create a worker per CPU core
  }
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died. Restarting...`);
    cluster.fork();
  });
} else {
  http.createServer((req, res) => {
    res.end('Hello from worker ' + process.pid);
  }).listen(3000);
}
```

**Use Cases:** CPU-bound workloads, maximizing multi-core utilization for HTTP servers.

### Q: What is the difference between Cluster and Worker Threads?

**Answer:**

| Feature | Cluster | Worker Threads |
|---|---|---|
| **Isolation** | Separate processes, separate memory | Same process, can share memory (`SharedArrayBuffer`) |
| **Use case** | Scaling HTTP servers across CPU cores | CPU-intensive tasks (image processing, crypto) |
| **Communication** | IPC (inter-process communication) | `MessagePort` (faster) |
| **Overhead** | Higher (separate V8 instances) | Lower (threads within same process) |

---

## 6 Buffers

### Q: What is a Buffer in Node.js?

**Answer:**

A Buffer is a **fixed-size chunk of memory** allocated outside the V8 heap, used for handling raw binary data (files, network packets, images).

```js
// Creating buffers
const buf1 = Buffer.alloc(10);          // 10 zero-filled bytes
const buf2 = Buffer.from('Hello');       // from string (UTF-8)
const buf3 = Buffer.from([0x48, 0x65]); // from byte array

// Converting
buf2.toString('utf-8');   // 'Hello'
buf2.toString('base64');  // 'SGVsbG8='

// Common operations
buf1.length;              // 10
Buffer.concat([buf2, buf3]);
buf2.slice(0, 3);         // 'Hel'
```

**Why not just use strings?** Strings are UTF-16 encoded in V8. Buffers let you work with arbitrary binary data (TCP streams, images, protobuf) without encoding overhead.

---

## 7 Express & Middleware

### Q: What is middleware in Express? Explain the execution order.

**Answer:**

Middleware functions have access to `req`, `res`, and `next`. They execute **in the order they are registered** and can modify the request/response, end the cycle, or pass control to the next middleware.

```js
// Application-level middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // MUST call next() or the request hangs
});

// Route-level middleware
app.get('/users', authenticate, authorize('admin'), (req, res) => {
  res.json(users);
});

// Error-handling middleware (4 arguments!)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.statusCode || 500).json({
    error: err.message || 'Internal Server Error'
  });
});
```

**Types of middleware:**
1. **Application-level** — `app.use()`, `app.get()`, etc.
2. **Router-level** — `router.use()`
3. **Error-handling** — Has 4 parameters: `(err, req, res, next)`
4. **Built-in** — `express.json()`, `express.static()`, `express.urlencoded()`
5. **Third-party** — `cors`, `helmet`, `morgan`

### Q: How does Express error handling work?

**Answer:**

```js
// Synchronous errors are caught automatically
app.get('/sync', (req, res) => {
  throw new Error('Sync error'); // Express catches this
});

// Async errors MUST be passed to next()
app.get('/async', async (req, res, next) => {
  try {
    const data = await fetchData();
    res.json(data);
  } catch (err) {
    next(err); // Pass to error-handling middleware
  }
});

// Express 5+ catches async errors automatically
// In Express 4, wrap with a helper:
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json(users);
}));
```

---

## 8 REST API Design

### Q: What are REST API design best practices?

**Answer:**

**URL Design:**
```
GET    /api/v1/users          — List users
GET    /api/v1/users/:id      — Get single user
POST   /api/v1/users          — Create user
PUT    /api/v1/users/:id      — Full update
PATCH  /api/v1/users/:id      — Partial update
DELETE /api/v1/users/:id      — Delete user
GET    /api/v1/users/:id/orders  — Nested resources
```

**Key Principles:**
- Use **nouns** for resources, not verbs (`/users` not `/getUsers`)
- Use **plural names** (`/users` not `/user`)
- **Version** your API (`/api/v1/`)
- Use proper **HTTP status codes**
- Support **pagination**, **filtering**, **sorting**
- Return **consistent** response structure

**Status Codes:**

| Code | Meaning | When to use |
|---|---|---|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation error |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 422 | Unprocessable Entity | Semantic validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server error |

**Pagination Example:**
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

---

## 9 Authentication — JWT, OAuth, Sessions

### Q: How does JWT authentication work?

**Answer:**

**JWT (JSON Web Token)** consists of three Base64-encoded parts separated by dots:
```
header.payload.signature
```

- **Header:** Algorithm & token type (`{"alg": "HS256", "typ": "JWT"}`)
- **Payload:** Claims — user data, expiration, issuer
- **Signature:** `HMACSHA256(base64(header) + "." + base64(payload), secret)`

**Flow:**
```
1. User logs in with credentials
2. Server verifies credentials, creates JWT with secret
3. Server sends JWT to client
4. Client stores JWT (httpOnly cookie or memory)
5. Client sends JWT in Authorization header: "Bearer <token>"
6. Server verifies signature and extracts user data
```

**Implementation:**
```js
const jwt = require('jsonwebtoken');

// Generate token
const token = jwt.sign(
  { userId: user._id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: '24h' }
);

// Verify middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token provided' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
};
```

### Q: JWT vs Sessions — tradeoffs?

**Answer:**

| Feature | JWT (Stateless) | Sessions (Stateful) |
|---|---|---|
| **Storage** | Client-side (cookie/memory) | Server-side (memory/Redis/DB) |
| **Scalability** | Easy — no server state | Requires shared session store |
| **Revocation** | Hard — must use blocklist | Easy — delete from store |
| **Size** | Larger (payload in token) | Small session ID |
| **Security** | Vulnerable if stored in localStorage (XSS) | CSRF protection needed |
| **Best for** | Microservices, APIs, mobile | Traditional web apps, monoliths |

### Q: What is the OAuth 2.0 flow?

**Answer:**

**Authorization Code Flow (most secure, for server-side apps):**
```
1. User clicks "Login with Google"
2. App redirects to Google's authorization endpoint
3. User consents → Google redirects back with an authorization CODE
4. App's backend exchanges code for access_token (server-to-server)
5. App uses access_token to fetch user info from Google
6. App creates session/JWT for the user
```

**Why not send the token directly?** The authorization code flow keeps the access token on the server side, never exposed to the browser.

---

## 10 Error Handling

### Q: What are the best practices for error handling in Node.js?

**Answer:**

**1. Custom Error Classes:**
```js
class AppError extends Error {
  constructor(message, statusCode, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational; // operational vs programming error
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404);
  }
}
```

**2. Global Handlers:**
```js
// Unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  // Log and then gracefully shutdown
  server.close(() => process.exit(1));
});

// Uncaught exceptions
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  // MUST exit — app is in an undefined state
  process.exit(1);
});
```

**3. Graceful Shutdown:**
```js
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Shutting down gracefully...');
  server.close(() => {
    // Close DB connections, flush logs
    mongoose.connection.close(false, () => {
      process.exit(0);
    });
  });
});
```

**Operational vs Programming Errors:**
- **Operational:** Expected failures (invalid input, DB timeout, network error). Handle gracefully.
- **Programming:** Bugs (TypeError, undefined access). Crash and restart.

---

## 11 Security

### Q: What are the common security vulnerabilities in Node.js applications?

**Answer:**

**1. SQL/NoSQL Injection:**
```js
// BAD — NoSQL injection
const user = await User.findOne({ email: req.body.email, password: req.body.password });
// Attacker sends: { "email": "admin@example.com", "password": { "$ne": "" } }

// GOOD — validate input
const { email, password } = req.body;
if (typeof email !== 'string' || typeof password !== 'string') {
  return res.status(400).json({ error: 'Invalid input' });
}
const user = await User.findOne({ email });
const isMatch = await bcrypt.compare(password, user.password);
```

**2. XSS (Cross-Site Scripting):**
```js
// Sanitize user input before storing/rendering
const sanitizeHtml = require('sanitize-html');
const clean = sanitizeHtml(userInput);
```

**3. CSRF (Cross-Site Request Forgery):**
```js
// Use csrf tokens or SameSite cookies
app.use(csrf({ cookie: true }));
// Set cookie: res.cookie('token', jwt, { httpOnly: true, secure: true, sameSite: 'Strict' })
```

**4. Security Headers (use Helmet):**
```js
const helmet = require('helmet');
app.use(helmet()); // Sets X-Frame-Options, CSP, HSTS, etc.
```

**5. Rate Limiting:**
```js
const rateLimit = require('express-rate-limit');
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: 'Too many requests'
}));
```

**6. Environment Variables:** Never hardcode secrets. Use `.env` files with `dotenv`.

---

## 12 Database — MongoDB & SQL

### Q: MongoDB schema design best practices?

**Answer:**

**Embedding vs Referencing:**

| Factor | Embed (denormalize) | Reference (normalize) |
|---|---|---|
| **When** | Data is read together often | Data is large or independent |
| **Example** | User → Address | User → Orders |
| **Performance** | Single read (fast) | Multiple reads (joins) |
| **Document size** | Must stay under 16MB | No limit concern |

**Indexing:**
```js
// Single field index
userSchema.index({ email: 1 });

// Compound index — order matters!
orderSchema.index({ userId: 1, createdAt: -1 });

// Text index for search
postSchema.index({ title: 'text', body: 'text' });

// TTL index for auto-expiry
sessionSchema.index({ createdAt: 1 }, { expireAfterSeconds: 3600 });
```

**Aggregation Pipeline:**
```js
const results = await Order.aggregate([
  { $match: { status: 'completed' } },
  { $group: { _id: '$userId', totalSpent: { $sum: '$amount' } } },
  { $sort: { totalSpent: -1 } },
  { $limit: 10 }
]);
```

### Q: Explain database transactions.

**Answer:**

```js
// MongoDB transactions (replica set required)
const session = await mongoose.startSession();
session.startTransaction();
try {
  await Account.updateOne({ _id: from }, { $inc: { balance: -amount } }, { session });
  await Account.updateOne({ _id: to }, { $inc: { balance: amount } }, { session });
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

---

## 13 Caching — Redis & Strategies

### Q: What caching strategies do you know?

**Answer:**

**1. Cache-Aside (Lazy Loading):**
```js
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await User.findById(id);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user)); // TTL: 1 hour
  return user;
}
```

**2. Write-Through:** Write to cache and DB simultaneously.

**3. Write-Behind (Write-Back):** Write to cache first, asynchronously write to DB.

**Cache Invalidation Strategies:**
- **TTL (Time-To-Live):** Auto-expire after a set time.
- **Event-based:** Invalidate on write/update operations.
- **Versioned keys:** `user:123:v2`

```js
// Invalidation on update
async function updateUser(id, data) {
  const user = await User.findByIdAndUpdate(id, data, { new: true });
  await redis.del(`user:${id}`); // Invalidate cache
  return user;
}
```

---

## 14 Memory Leaks & Profiling

### Q: How do you detect and fix memory leaks in Node.js?

**Answer:**

**Common Causes:**
1. **Global variables** accumulating data
2. **Closures** retaining references
3. **Event listeners** not being removed
4. **Unbounded caches** (Maps/objects growing forever)
5. **Unfinished timers** (`setInterval` without `clearInterval`)

**Detection:**
```js
// Monitor memory usage
setInterval(() => {
  const usage = process.memoryUsage();
  console.log({
    rss: `${(usage.rss / 1024 / 1024).toFixed(2)} MB`,      // Total memory
    heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
    heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
    external: `${(usage.external / 1024 / 1024).toFixed(2)} MB`,
  });
}, 5000);
```

**Tools:**
- `--inspect` flag + Chrome DevTools (heap snapshots)
- `clinic.js` — flamegraphs, bottleneck detection
- `node --prof` — V8 profiler

**Fix Patterns:**
```js
// BAD — event listener leak
class Emitter extends EventEmitter {}
const emitter = new Emitter();
function handler() { /* ... */ }
emitter.on('data', handler);
// Over time, many listeners accumulate

// GOOD — remove listeners
emitter.off('data', handler);
// Or use once()
emitter.once('data', handler);
```

---

## 15 Microservices

### Q: What patterns are used in microservices with Node.js?

**Answer:**

**Communication Patterns:**
1. **Synchronous:** HTTP/REST, gRPC (faster, binary)
2. **Asynchronous:** Message queues (RabbitMQ, Kafka, Redis pub/sub)

**Key Patterns:**
- **API Gateway:** Single entry point, routes to services, handles auth, rate limiting
- **Circuit Breaker:** Prevents cascading failures by stopping calls to failing services
- **Saga Pattern:** Manages distributed transactions across services
- **Event Sourcing:** Store state changes as events instead of current state
- **CQRS:** Separate read and write models

**Circuit Breaker Example:**
```js
const CircuitBreaker = require('opossum');

const options = {
  timeout: 3000,      // 3s timeout
  errorThresholdPercentage: 50,  // Open circuit at 50% error rate
  resetTimeout: 10000  // Try again after 10s
};

const breaker = new CircuitBreaker(callExternalService, options);

breaker.fire(params)
  .then(result => res.json(result))
  .catch(err => res.status(503).json({ error: 'Service unavailable' }));

breaker.on('open', () => console.log('Circuit opened!'));
breaker.on('halfOpen', () => console.log('Circuit half-open, testing...'));
breaker.on('close', () => console.log('Circuit closed, all good'));
```

---

## 16 Logging & Monitoring

### Q: What are best practices for logging in production Node.js apps?

**Answer:**

**Structured Logging:**
```js
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Usage
logger.info('User logged in', { userId: user.id, ip: req.ip });
logger.error('Payment failed', { orderId, error: err.message, stack: err.stack });
```

**Best Practices:**
- Use **structured JSON** logs (not `console.log` strings)
- Include **correlation IDs** for request tracing across services
- Log at appropriate **levels**: error > warn > info > debug
- Never log **sensitive data** (passwords, tokens, PII)
- Use **log aggregation** tools (ELK Stack, Datadog, CloudWatch)

---

## 17 Docker & Deployment

### Q: Write an optimized Dockerfile for a Node.js app.

**Answer:**

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app

# Security: run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /app/node_modules ./node_modules
COPY . .

USER appuser
EXPOSE 3000

# Use dumb-init to handle signals properly
RUN apk add --no-cache dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server.js"]
```

**Key Optimizations:**
- **Multi-stage build** — smaller final image
- **`npm ci`** — deterministic installs from lockfile
- **Alpine base** — smaller image (~50MB vs ~350MB)
- **Non-root user** — security best practice
- **`.dockerignore`** — exclude `node_modules`, `.git`, `.env`
- **`dumb-init`** — properly forwards signals (SIGTERM) for graceful shutdown

---

## 18 Design Patterns in Node.js

### Q: What design patterns are commonly used in Node.js?

**Answer:**

**1. Singleton:**
```js
// Node modules are cached, so this is a natural singleton
class Database {
  constructor() {
    if (Database.instance) return Database.instance;
    this.connection = null;
    Database.instance = this;
  }
  async connect(uri) {
    this.connection = await mongoose.connect(uri);
  }
}
module.exports = new Database();
```

**2. Observer (EventEmitter):**
```js
const EventEmitter = require('events');
class OrderService extends EventEmitter {
  async createOrder(data) {
    const order = await Order.create(data);
    this.emit('orderCreated', order);
    return order;
  }
}
// Listeners
orderService.on('orderCreated', sendEmail);
orderService.on('orderCreated', updateInventory);
```

**3. Factory:**
```js
class NotificationFactory {
  static create(type) {
    switch (type) {
      case 'email': return new EmailNotification();
      case 'sms': return new SMSNotification();
      case 'push': return new PushNotification();
      default: throw new Error(`Unknown type: ${type}`);
    }
  }
}
```

**4. Middleware / Chain of Responsibility:**
```js
// Express middleware IS the chain of responsibility pattern
app.use(authenticate);
app.use(validate);
app.use(rateLimiter);
app.use(controller);
```

**5. Repository Pattern:**
```js
class UserRepository {
  async findById(id) { return User.findById(id); }
  async create(data) { return User.create(data); }
  async update(id, data) { return User.findByIdAndUpdate(id, data, { new: true }); }
  async delete(id) { return User.findByIdAndDelete(id); }
}
// Controller uses repository, not model directly
// Makes it easy to swap DB or add caching
```

---

> **Tip:** For senior-level interviews, focus on **Event Loop internals, Streams with backpressure, error-handling architecture, caching strategies, and system design** — these separate experienced developers from beginners.
