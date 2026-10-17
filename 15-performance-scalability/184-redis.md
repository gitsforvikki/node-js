# Lesson 184 — Redis in Node.js

## 1. What is Redis?
**Redis** is an in-memory data store commonly used as a **shared cache**, rate-limit counter, session store, and lightweight messaging/queue building block (depending on guarantees needed).

Unlike a JavaScript `Map` inside one Node process, a Redis server can be shared by **multiple app instances**.

```text
Node API A  --\
             -> Redis (shared cache)
Node API B  --/       |
                     MongoDB remains source of truth
```

## 2. Start locally
With Docker installed:
```bash
docker run --rm --name redis-demo -p 6379:6379 redis:7
```
Redis in this demo is a local development instance. Do not publicly expose unauthenticated Redis; production needs private networking, authentication, TLS where appropriate, and access controls.

## 3. Connect from Node.js
```bash
npm install redis
```
```js
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL ?? "redis://127.0.0.1:6379",
});
redis.on("error", (err) => console.error("Redis error:", err.message));
await redis.connect();

await redis.set("greeting", "hello", { EX: 60 });
console.log(await redis.get("greeting")); // "hello"
await redis.quit();
```
`EX: 60` means expire after **60 seconds**.

## 4. Cache-aside product endpoint
Assume `redis` is connected and `Product` is your Mongoose model:
```js
app.get("/products/:id", async (req, res) => {
  const key = `product:${req.params.id}`;
  const cached = await redis.get(key);

  if (cached) return res.json(JSON.parse(cached));

  const product = await Product.findById(req.params.id)
    .select("name price").lean();
  if (!product) return res.sendStatus(404);

  await redis.set(key, JSON.stringify(product), { EX: 60 });
  res.json(product);
});
```
**Production additions:** validate ObjectId first; authorize private data; handle Redis outages using a bounded DB fallback; invalidate `product:id` after updates; consider concurrent cache misses. Don't expose raw errors or treat cache data as authoritative.

## 5. Redis data types
| Type | Common use |
| --- | --- |
| String | JSON cache or counters |
| Hash | Small fields of an object |
| List | Simple ordered values |
| Set | Unique membership |
| Sorted set | Rankings / time-ordered scores |
| Stream | Append-only events / consumers |

Redis is not merely a cache: it can have persistence options, but guarantees and use cases depend on deployment. Avoid treating transient caching as a durable job system without understanding delivery guarantees.

## 6. Rate-limiting counter idea
```js
const key = `attempts:${accountId}`;
const attempts = await redis.incr(key);
if (attempts === 1) await redis.expire(key, 60);
```
This illustrates `INCR` and `EXPIRE`, but **the two commands are not atomic together**. For production use a tested Redis-backed rate limiter or an atomic Lua/script/transaction design with correct expiry, concurrency and failover behavior.

## 7. Redis and sessions
A shared session store allows users to hit API A then API B without losing server-side sessions. Protect session cookies and TTLs; session storage and rate limiting require careful consistency and security.

## 8. Redis failure planning
Decide per feature:
- Cached product list: often fall back to MongoDB (with request limits).
- Security rate limits: fail-open vs fail-closed requires risk analysis.
- Session store: loss of availability may require rejecting authenticated requests.

Monitor memory, hit rate, evictions, latency, rejected connections and expiration behavior.

## 9. Common mistakes
Putting passwords in Redis as plain cache values; no TTL; treating Redis as unlimited memory; exposing Redis publicly; using a local in-process Map across multiple replicas; forgetting invalidation or failure handling.

## Practice
Cache `GET /products/:id` for 60 seconds, read the key with Redis CLI, update the product and add invalidation. Compare first-request vs cached-request latency.

**Interview answer:** Redis is a fast shared in-memory store used for caching, sessions and counters. Design TTLs, key structure, consistency, authorization and outage behavior; MongoDB or another primary database usually remains the source of truth.
