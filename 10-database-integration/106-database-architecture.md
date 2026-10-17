# Lesson 106 — Database Architecture in Node.js

## Why this lesson matters

Most Node.js backend applications are not limited by route syntax.

They are limited by how well they interact with data.

A strong backend engineer should understand:

- where database logic belongs
- how connections are managed
- why connection pools matter
- how transactions fit into services
- how repositories separate data access
- how to avoid leaking database details into the HTTP layer

---

## 1. Database integration is more than "connect and query"

**Simple explanation:** A route should not need to know every database detail. As a project grows, separate the HTTP part, business rules, and queries. For a small CRUD app, fewer layers can still be fine.

```text
GET /users/123 -> Route -> Controller -> Service -> Repository -> MongoDB
```


Weak architecture:

```text
Route
  |
  v
direct DB query
  |
  v
response
```

Better:

```text
Route
  |
  v
Controller
  |
  v
Service
  |
  v
Repository / Data Access Layer
  |
  v
Database
```

---

## 2. Controller responsibility

**In simple words:** A controller reads `req.params`, `req.query`, or `req.body`, calls a service, and chooses the HTTP response. It should not contain the main business rules.

**Express 5 version:** Async handler errors are forwarded automatically to error middleware; a `try/catch` that only calls `next(error)` is not required.

```js
// controllers/user.controller.js
export async function getUserController(req, res) {
  const user = await userService.getUserById(req.params.id);
  return res.status(200).json({ success: true, data: user });
}
```

The service handles the missing-user case; error middleware turns that into an HTTP response.


Controller should handle HTTP concerns:

- params
- query
- body
- status code
- response

Example:

```js
export async function getUser(
  req,
  res,
  next
) {
  try {
    const user =
      await userService.getById(
        req.params.id
      );

    res.json(user);
  } catch (error) {
    next(error);
  }
}
```

---

## 3. Service responsibility

**In simple words:** A service decides **what should happen**, not how to send an HTTP response.

```js
// services/user.service.js
export async function getUserById(id) {
  const user = await userRepository.findById(id);
  if (!user) throw new AppError("User not found", 404);
  return user;
}
```

Flow: ask repository for the user -> check the business condition -> return data or throw an application error. `AppError` here means your own error class.


Service contains business logic.

Example:

```js
export async function getUserById(
  id
) {
  const user =
    await userRepository.findById(
      id
    );

  if (!user) {
    throw new UserNotFoundError();
  }

  return user;
}
```

---

## 4. Repository responsibility

**In simple words:** A repository knows **how to fetch or save data**.

```js
// repositories/user.repository.js
import User from "../models/user.model.js";

export function findById(id) {
  return User.findById(id); // Mongoose query (awaitable)
}
```

The service calls `findById(id)` without writing a MongoDB query itself. Replacing MongoDB with SQL would mainly change this data-access code, although database-specific behavior may still require other changes.


Repository hides data-access details.

Example:

```js
export function findById(
  id
) {
  return User.findById(id);
}
```

The service should not need to know whether storage uses:

- MongoDB
- PostgreSQL
- Redis
- external API

---

## 5. Why repositories help

**Example:** If `profileService` and `orderService` both need to look up users, they can share `userRepository.findById(id)` instead of copying the same query.

**Do not over-engineer:** For a tiny app with simple CRUD, directly calling Mongoose from a service may be easier. Add a repository when it helps testing, reuse, or separating database-specific logic.


Benefits:

- cleaner services
- centralized query logic
- easier testing
- easier DB migration
- reduced ORM leakage

But:

> Do not add a repository layer only for ceremony.

If it adds no abstraction value, keep architecture proportionate.

---

## 6. Connection lifecycle

**Connection lifecycle in simple steps:** load configuration -> check required settings -> connect -> begin accepting requests -> handle disconnections/shutdown.

```text
Config -> DB connection ready -> app.listen() -> requests
```

This reduces startup requests failing because the database is not connected. Later connection loss still needs monitoring and error handling.


A production app should usually connect before accepting traffic.

```text
load config
   |
   v
connect database
   |
   v
verify ready
   |
   v
start HTTP server
```

Bad:

```text
listen first
   |
   v
DB still connecting
   |
   v
requests fail randomly
```

---

## 7. Startup example

**Runnable-style Express + Mongoose startup:**

```js
import mongoose from "mongoose";
import app from "./app.js";

async function startServer() {
  try {
    if (!process.env.MONGODB_URI) throw new Error("MONGODB_URI missing");
    await mongoose.connect(process.env.MONGODB_URI);
    console.log("Database connected");
    app.listen(3000, () => console.log("Server running on 3000"));
  } catch (error) {
    console.error("Startup failed:", error);
    process.exit(1);
  }
}

startServer();
```

`app.js` exports the Express app; load environment variables before running this code.


```js
async function start() {
  await connectDatabase();

  app.listen(PORT);
}

start().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

---

## 8. Connection pooling

**Think of a pool as reusable database connections.** For example, 100 incoming requests can share a pool of 10 connections over time. At most the configured number of pool connections can perform operations concurrently on those connections; other work may wait.

```text
100 requests -> Node.js -> Pool [1,2,3,...10] -> Database
```

A pool saves the repeated cost of creating a new connection for each request.


Relational databases commonly use connection pools.

Mental model:

```text
Node App
   |
   v
Connection Pool
   |
   +--> DB connection 1
   +--> DB connection 2
   +--> DB connection 3
   +--> DB connection 4
```

Instead of opening a new DB connection for every request.

---

## 9. Why not create one connection per request?

**Why opening connections is expensive:** Each new connection may require TCP/TLS setup, authentication, and database resources. Creating one per HTTP request wastes time and can hit server limits.

**Prefer:** initialize one shared database client at startup, then reuse its managed pool for queries.


Because connection establishment can be expensive:

- authentication
- TCP/TLS setup
- DB process/thread resources
- memory
- server connection limits

A pool reuses existing connections.

---

## 10. MongoDB connection pooling

**Mongoose example:**

```js
await mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 10,
});
```

This configures the maximum size of the main connection pool (the driver may create extra monitoring connections). Avoid `mongoose.connect()` inside a route handler. One shared Mongoose connection can manage multiple physical connections.


MongoDB drivers also maintain internal pools.

You usually create a shared client/connection rather than reconnecting for each request.

Bad:

```js
app.get("/users", async () => {
  await mongoose.connect(...);
});
```

Very bad architecture.

---

## 11. Singleton-style DB client

**Singleton-style means shared client, not one physical socket.**

```js
// config/database.js
import mongoose from "mongoose";

export function connectDB() {
  return mongoose.connect(process.env.MONGODB_URI);
}

// server.js: await connectDB() once at startup
// controllers/services use models through that shared connection
```

In serverless environments, connection reuse needs extra care because instances may start and stop frequently.


Typical idea:

```text
application process
      |
      v
one shared DB client
      |
      v
internal connection pool
```

This is normal and efficient.

---

## 12. Configuration

**Example `.env` values (use real secrets only in your private environment):**

```dotenv
MONGODB_URI=mongodb://127.0.0.1:27017/myapp
PORT=3000
```

Load environment variables before reading `process.env` (for example, Node's `--env-file` option or `dotenv`). Never commit real credentials to Git.


Keep credentials in environment configuration.

```text
DATABASE_URL
MONGODB_URI
```

Do not hardcode secrets in source.

---

## 13. Validate config at startup

**Fail fast with a useful error:**

```js
const uri = process.env.MONGODB_URI;
if (!uri) {
  throw new Error("Missing MONGODB_URI environment variable");
}
```

Checking at startup gives an immediate, understandable failure instead of a mysterious error during the first user request. For bigger projects, a schema validator can check all settings together.


If `DATABASE_URL` is missing, fail fast.

Bad:

```text
app starts
   |
   v
first DB request
   |
   v
mysterious connection failure
```

Better:

```text
startup validates config
   |
   v
fail immediately
```

---

## 14. Health vs readiness

**Example:** Express can be running while MongoDB is unavailable. That process is *alive*, but it may not be *ready* for DB-dependent traffic.

```js
import mongoose from "mongoose";

app.get("/health", (req, res) => res.json({ alive: true }));
app.get("/ready", (req, res) => {
  const ready = mongoose.connection.readyState === 1;
  res.status(ready ? 200 : 503).json({ ready });
});
```

`readyState` is a basic check, not a full database round-trip test. More robust readiness checks may test important dependencies.


Health:

```text
process is alive
```

Readiness:

```text
process can actually serve traffic
```

A service may be alive but not ready because DB is unavailable.

---

## 15. Transactions belong around business workflows

**Example: order creation.** Suppose you must create an order and reduce product stock together. If stock update fails, you do not want the order saved alone.

```js
await mongoose.connection.transaction(async (session) => {
  const updated = await Product.updateOne(
    { _id: productId, stock: { $gte: quantity } },
    { $inc: { stock: -quantity } },
    { session }
  );
  if (updated.modifiedCount !== 1) throw new Error("Out of stock");

  await Order.create([{ userId, productId, quantity }], { session });
});
```

Either the database changes commit together or the transaction aborts. MongoDB transactions require a supported deployment (such as a replica set). External payment API calls cannot be rolled back by a database transaction; use a separate payment workflow.


Example:

```text
Create Order
   |
   +--> decrease stock
   +--> create order
   +--> create payment record
```

If all must succeed together, transaction boundary belongs in business logic/service layer.

---

## 16. Avoid transaction logic in controllers

**Keep transaction decisions in the service.**

```js
// Controller: HTTP only
export async function createOrderController(req, res) {
  const order = await orderService.createOrder(req.body);
  return res.status(201).json({ success: true, data: order });
}

// Service: validates stock and manages the DB transaction
```

This lets the same order workflow be called from HTTP, jobs, or tests without copying transaction logic.


Bad:

```js
app.post("/orders", async (req, res) => {
  const tx = ...
  // business transaction logic here
});
```

Better:

```js
await orderService.createOrder(...);
```

Service controls transaction.

---

## 17. Query efficiency matters

**A correct query can still be slow.** Think about indexes, pagination, unnecessary `populate()` calls, and fetching too many rows.

```js
// Often useful for searching by email:
userSchema.index({ email: 1 });

// Avoid returning every document in a large collection:
const users = await User.find().limit(20);
```

Choose indexes based on actual query patterns and use `explain()` to inspect plans when needed. Indexes also cost space and slow down some writes.


A working query is not necessarily a good query.

You must think about:

- indexes
- selected fields
- joins/populate
- pagination
- N+1 queries
- aggregation
- query plans

---

## 18. Select only what you need

**Only fetch fields the client needs.**

```js
// Less efficient if you only need name and email:
const allFields = await User.find();

// Better for that requirement:
const users = await User.find().select("name email");
```

For SQL, prefer `SELECT id, name, email FROM users` instead of `SELECT *` when only those columns are needed. Smaller results use less network bandwidth and memory.


Bad:

```text
SELECT *
```

when client needs only:

```text
id
name
email
```

Likewise in MongoDB:

```js
User.find()
  .select("name email");
```

Less data means:
- less DB work
- less network
- less memory

---

## 19. Data-access errors

**Never send raw database error details directly to clients.** Log technical details on the server; send a safe message and appropriate status code.

```js
// Express error middleware (must have four parameters)
app.use((err, req, res, next) => {
  console.error(err);
  const status = err.statusCode ?? 500;
  res.status(status).json({
    success: false,
    message: status >= 500 ? "Internal server error" : err.message,
  });
});
```

Map known errors (such as duplicate email or invalid input) to meaningful application errors before responding. Do not assume every raw DB error is safe to expose.


Raw DB errors should not leak directly.

Architecture:

```text
Mongo/Postgres error
      |
      v
Repository
      |
      v
Domain/Application Error
      |
      v
Error Middleware
```

---

## 20. Retry caution

**Retry only when it is safe.** A short network failure may be temporary. But retrying an order/payment write after an unknown result can create duplicates.

```text
Read fails temporarily -> a limited retry may help
Payment succeeds but response is lost -> blind retry may charge twice
```

Use bounded retries, backoff, and idempotency keys or unique constraints for important writes. Check whether your driver already retries certain operations.


Do not retry every DB failure blindly.

Retry may be appropriate for:
- transient network failures
- selected deadlock/serialization errors

Dangerous for:
- non-idempotent writes
- unknown transaction state

Retries must understand operation semantics.

---

## 21. Database timeout

**Example: put a limit on MongoDB query execution.**

```js
const users = await User.find().maxTimeMS(5000);
```

This requests a roughly 5-second **server-side execution limit**, not a guarantee that the entire network request finishes within 5 seconds. Production apps may also need driver timeouts and cancellation.


Queries should not run forever.

Use:
- driver timeout
- statement timeout
- max execution time
- request cancellation where supported

---

## 22. Backpressure and DB

**Database backpressure is a similar capacity problem to stream backpressure.**

```text
1000 requests -> pool with 20 connections -> requests wait
```

If queries arrive faster than the database completes them, waiting work grows and latency rises. Limit concurrency, optimize queries, use sensible queue/request limits, and return a controlled error when overloaded. A connection pool alone does not make unlimited traffic safe.


Your API can accept requests faster than DB can process them.

If DB pool is saturated:

```text
1000 requests
   |
   v
pool of 20 connections
   |
   v
980 wait
```

This increases latency.

Rate limiting, queueing, pool sizing, and query optimization all matter.

---

## 23. Pool size is not "bigger is always better"

**Example across instances:**

```text
20 Node.js instances x 50 connections each = up to 1000 pool connections
```

A bigger pool is not automatically faster: too many connections can consume DB memory and increase contention. Size pools using database capacity, application instance count, and real performance measurements.


Too many connections can overwhelm DB.

Example:

```text
20 app instances
x 50 connections
= 1000 DB connections
```

Database may not handle that efficiently.

Capacity planning matters.

---

## 24. Read/write separation

**At scale, writes often go to the primary and some reads to replicas.**

```text
Write -> Primary -> replication -> Replica <- Read
```

**Catch:** replicas may lag. Immediately after a profile update, reading from a replica could show the old profile. For read-after-write consistency, route sensitive reads appropriately or use the database's consistency options.


At larger scale:

```text
Writes
  -> primary

Reads
  -> replica
```

This introduces replication lag.

A user may write data and immediately read stale data from a replica.

Know the consistency trade-off.

---

## 25. Caching

**Cache example:**

```text
GET /products/123 -> Redis cache
  cache hit  -> return cached product
  cache miss -> query MongoDB -> cache result -> return
```

Caching can reduce DB load, but stale values are possible. On a product update, expire or invalidate the cached product. Add caching when measurements show repeated reads are worth the extra complexity.


Not every request should always hit DB.

Possible flow:

```text
request
   |
   v
cache
   |
   +--> hit -> response
   |
   +--> miss -> DB -> cache -> response
```

But caching adds invalidation complexity.

---

## 26. ORM/ODM vs raw queries

**Compare database access styles:**

```js
// ODM: Mongoose (MongoDB)
const user = await User.findById(id);

// Raw SQL: node-postgres (PostgreSQL)
const result = await pool.query(
  "SELECT id, name FROM users WHERE id = $1",
  [id]
);
```

Mongoose is an **ODM** for documents; tools such as Drizzle and Prisma are used for SQL-oriented data access. Raw SQL offers direct control, while an ORM/ODM offers models and helpers. Parameterize SQL values instead of building queries by string concatenation.


ORM/ODM benefits:

- productivity
- schema/model abstraction
- validation
- query helpers

Raw query benefits:

- full control
- sometimes clearer performance behavior
- fewer abstraction surprises

Choose intentionally.

---

## 27. Common mistakes

**How to avoid these mistakes:** connect at startup, keep HTTP logic out of database code, index frequent queries, paginate large result sets, retry only safe operations, measure pool sizing, and convert internal DB failures into safe application errors.


### Mistake 1
Connecting to DB per request.

### Mistake 2
DB logic in controllers.

### Mistake 3
No indexes.

### Mistake 4
Huge unbounded queries.

### Mistake 5
Blind retries.

### Mistake 6
Oversized connection pools.

### Mistake 7
Leaking raw DB errors.

---

## 28. Interview questions

**Practice answering in plain English:**

- **Repository:** “It keeps queries in one place, away from business rules.”
- **Pool:** “It reuses connections instead of opening one for every request.”
- **Transaction:** “It groups related DB changes so they succeed or fail together.”
- **Startup connection:** “It prevents the server from accepting DB requests before it is ready.”


### Why use a repository layer?

To isolate data-access concerns and keep business logic independent from DB implementation details.

### Why use connection pooling?

To reuse expensive DB connections and control concurrency.

### Where should transactions live?

Usually around business workflows in the service/application layer.

### Why not connect on every request?

Connection setup is expensive and wastes DB resources.

---

## 29. Strong interview answer

**Shorter interview version:** “I keep HTTP handling in controllers, business rules in services, and database queries in repositories when useful. I reuse pooled connections, use transactions for related writes, and pay attention to indexes, errors, and timeouts.”


> I keep database access behind a repository or data-access boundary, while services own business logic and transaction boundaries. The application establishes shared database connectivity before accepting traffic and uses pooling rather than creating connections per request. I also control query size, use proper indexes, map raw DB errors into application errors, and size the pool based on total application instances and database capacity rather than simply increasing it.

---

## Interview-Ready Summary

```text
HTTP
  |
  v
Controller
  |
  v
Service
  |
  v
Repository
  |
  v
Database

Production concerns:
pooling
transactions
timeouts
indexes
query size
readiness
error mapping
```

## Practice Task

Design database architecture for an e-commerce API with:

- users
- products
- orders
- payments

Show:
- service boundaries
- repository boundaries
- transaction boundary
- connection lifecycle
