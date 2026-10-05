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

Because connection establishment can be expensive:

- authentication
- TCP/TLS setup
- DB process/thread resources
- memory
- server connection limits

A pool reuses existing connections.

---

## 10. MongoDB connection pooling

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

Keep credentials in environment configuration.

```text
DATABASE_URL
MONGODB_URI
```

Do not hardcode secrets in source.

---

## 13. Validate config at startup

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

Queries should not run forever.

Use:
- driver timeout
- statement timeout
- max execution time
- request cancellation where supported

---

## 22. Backpressure and DB

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
