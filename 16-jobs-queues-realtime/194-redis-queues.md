# Lesson 194 — Redis-based Queues

## 1. Why Redis for jobs?
Redis is a fast shared data store. A queue library can use Redis to coordinate producers and workers across Node.js processes.

```text
Express API ---> Redis queue <--- Worker process
  (producer)       (waiting)        (consumer)
```

## 2. Redis lists: concept, not complete production queue
`LPUSH jobs payload` adds a job; `BRPOP jobs 0` waits and removes one. But **remove-then-process** can lose a job if the worker crashes after removing it.

```text
BRPOP removes job -> worker crashes -> message gone
```
Do not build a critical queue solely from simple `LPUSH/BRPOP`.

## 3. Safer patterns
- **Redis Streams + consumer groups:** track pending entries and acknowledge after processing; reclaim pending work after failures.
- **BullMQ:** queue library that manages job state, delayed jobs, locks, retries and workers using Redis.
- **Redis Pub/Sub:** immediate broadcasts only; messages sent while subscriber offline are lost. Not a durable job queue.

## 4. Runnable Redis Stream producer/consumer basics
Prerequisites: Redis running; `npm i redis`.

**producer.js**
```js
import { createClient } from "redis";
const redis = createClient();
redis.on("error", console.error);
await redis.connect();

const id = await redis.xAdd("order-events", "*", {
  type: "order-created",
  orderId: "ord-123",
});
console.log("Added", id);
await redis.quit();
```

**consumer.js** (demonstration of one group consumer):
```js
import { createClient } from "redis";
const redis = createClient();
redis.on("error", console.error);
await redis.connect();

try {
  await redis.xGroupCreate("order-events", "emails", "0", { MKSTREAM: true });
} catch (err) {
  if (!String(err.message).includes("BUSYGROUP")) throw err;
}

const data = await redis.xReadGroup(
  "emails",
  "worker-1",
  [{ key: "order-events", id: ">" }],
  { COUNT: 10, BLOCK: 2000 }
);

for (const stream of data ?? []) {
  for (const entry of stream.messages) {
    console.log("Process:", entry.message);
    // ONLY acknowledge AFTER the business operation succeeds.
    await redis.xAck("order-events", "emails", entry.id);
  }
}
await redis.quit();
```

For production, add a long-running loop, schema validation, retry/reclaim of pending entries (e.g. XAUTOCLAIM), trimming/retention, idempotency, and shutdown handling. This is intentionally a **single-pass educational demonstration**.

## 5. Stream semantics
- `XADD`: append an event with ID.
- `XGROUP CREATE`: create consumer group.
- `XREADGROUP`: read via group.
- `XACK`: acknowledge processing.
- `XPENDING` / `XAUTOCLAIM`: inspect/reclaim jobs stuck with failed consumers.

Streams persist entries according to Redis persistence/retention configuration; Redis is not automatically an unlimited durable archive.

## 6. Why BullMQ for ordinary Node APIs?
A custom Streams queue requires handling retries, locking, concurrency, poison jobs, backoff, and scheduling. BullMQ packages many of these into familiar APIs; learn Streams to understand the mechanism, then use a maintained queue library for typical applications.

## 7. Common mistakes
Confusing Pub/Sub with queues; acknowledging before work finishes; never reclaiming pending jobs; forgetting memory/retention limits; assuming Redis has durable persistence regardless of configuration.

## Practice
Send three stream events; process and acknowledge them; then modify consumer to crash before XACK and inspect pending entries.

**Interview answer:** Redis supports queue-like patterns using lists or Streams, but reliability depends on acknowledgment, recovery, persistence and retention. BullMQ builds job-processing abstractions on Redis.
