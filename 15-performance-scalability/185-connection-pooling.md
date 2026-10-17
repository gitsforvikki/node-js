# Lesson 185 — Database Connection Pooling

## 1. What is a connection pool?
Opening a new database connection can involve TCP/TLS setup and authentication. **Pooling** means maintaining reusable connections so many requests share them instead of reconnecting on every API call.

Think of taxis: a company maintains several available taxis rather than buying a new taxi for every customer.

```text
100 HTTP requests
      |
      v
Mongoose shared MongoDB client
      |
      v
Pool of reusable connections
      |
      v
MongoDB server
```

## 2. Why it helps
- Reduces connection setup overhead.
- Caps concurrent connections per pool to avoid overwhelming the DB.
- Allows requests to wait for free connections.
- Provides settings for timeouts and idle connections.

**Important:** pool size does not equal request concurrency; async requests may be waiting on other tasks. A pool of 10 does not imply your API can serve only 10 HTTP clients.

## 3. Mongoose example
```js
import mongoose from "mongoose";

await mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 10,
  minPoolSize: 0,
  waitQueueTimeoutMS: 5000,
  serverSelectionTimeoutMS: 5000,
});

app.listen(3000);
```
Initialize **once during app startup**, not inside each route handler. The underlying MongoDB driver maintains connection pools **per server**; replica sets and monitoring may require additional connections. `maxPoolSize` is an upper limit, not a guarantee of exactly 10 open sockets.

## 4. Bad and good patterns
Bad:
```js
app.get("/users", async (req, res) => {
  await mongoose.connect(process.env.MONGODB_URI); // Don't reconnect per request
  res.json(await User.find());
});
```
Good:
```js
// startup: await mongoose.connect(...)
// route:
app.get("/users", async (req, res) => {
  const users = await User.find().limit(50).lean();
  res.json(users);
});
```

## 5. PostgreSQL connection pool example
```bash
npm install pg
```
```js
import pg from "pg";
const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});

const result = await pool.query("SELECT id, name FROM users LIMIT $1", [20]);
console.log(result.rows);
```
For a transaction, acquire **one client** using `pool.connect()`, use the same client throughout, then `client.release()` in `finally`. Multiple `pool.query()` calls may use different clients and must not be used as one transaction.

## 6. Why not make pool huge?
Imagine 8 Node replicas each allowing 50 database connections: capacity can approach **400 connections**, plus database-driver extras. Every connection uses DB resources; an oversized pool can increase contention rather than improve speed.

A rough planning starting point is: app replicas × max connections per replica, adjusted for DB topology, admin connections, monitoring and workload. **Benchmark and size for your database service limits.**

## 7. Queueing and timeouts
If all pool connections are busy, new DB operations may wait. Apply queue/wait timeouts and monitor checkout/connection-acquisition wait time. Distinguish:
- connection acquisition delay;
- query execution latency;
- overall HTTP latency.

Increasing pool size does not fix inefficient queries or missing indexes.

## 8. Serverless caution
Many short-lived serverless instances can create connection bursts. Follow provider-specific guidance, reuse clients within a warm instance and consider managed poolers or serverless-compatible drivers.

## 9. Common mistakes
Calling `connect()` inside controllers; forgetting to release manually acquired `pg` clients; making pool sizes huge; treating HTTP requests as DB connections; opening extra pools in every module.

## Practice
Measure queries with `maxPoolSize=5` vs `20` on your test DB under controlled load. Explain why results depend on workload, DB limits and query times.

**Interview answer:** Connection pooling reuses DB connections to reduce setup overhead and control simultaneous DB work. Create shared clients at startup, size pools across replicas, monitor waits, and release acquired connections properly.
