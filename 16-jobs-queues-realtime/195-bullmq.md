# Lesson 195 — BullMQ

## 1. What is BullMQ?
**BullMQ** is a Node.js job queue built on Redis. It helps enqueue work, process it with workers, configure retries/backoff and monitor completion/failure.

Example: An order API saves a payment receipt request; an independent worker sends the receipt.

## 2. Setup
```bash
npm install bullmq ioredis express
docker run --rm -d --name bullmq-redis -p 6379:6379 redis:7
```
Use `"type": "module"` for ESM. Put example files in the same directory.

## 3. Producer: add jobs
**queue.js**
```js
import { Queue } from "bullmq";
export const receiptQueue = new Queue("receipts", {
  connection: { host: "127.0.0.1", port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 1000 },
    removeOnComplete: { count: 1000 },
    removeOnFail: { count: 5000 },
  },
});
```
**api.js**
```js
import express from "express";
import { receiptQueue } from "./queue.js";
const app = express();
app.use(express.json());

app.post("/orders/:id/receipt", async (req, res) => {
  // Authenticate, authorize order, validate ID and limit requests first.
  const job = await receiptQueue.add("send-receipt", {
    orderId: req.params.id,
  });
  res.status(202).json({ jobId: job.id, state: "queued" });
});
app.listen(3000, () => console.log("API on 3000"));
```

## 4. Consumer: process jobs
**worker.js**
```js
import { Worker } from "bullmq";

const worker = new Worker("receipts", async (job) => {
  if (job.name !== "send-receipt") throw new Error("Unknown job");
  const { orderId } = job.data;
  if (typeof orderId !== "string") throw new Error("Invalid orderId");
  console.log("Send receipt for", orderId);
  // Real: fetch order + send email, then record delivery in DB.
  return { sent: true };
}, {
  connection: { host: "127.0.0.1", port: 6379 },
  concurrency: 5,
});
worker.on("completed", (job) => console.log("Completed", job.id));
worker.on("failed", (job, err) => console.error("Failed", job?.id, err.message));
```

Run `node worker.js` and `node api.js` separately. `curl -X POST http://localhost:3000/orders/ord-1/receipt` queues the example.

## 5. Job states and flow
```text
Queue.add() -> waiting -> active (worker holds lock)
                    |             |
                    |           success -> completed
                    |           failure -> delayed retry / failed
                    +--> delayed / scheduled
```
BullMQ may mark stalled jobs and retry them if a worker dies. **Do not assume exactly-once processing**.

## 6. Worker concurrency vs parallel CPU work
`concurrency: 5` means a worker can process multiple Promise-based jobs concurrently. It is useful when jobs mostly await email/network I/O. It **does not** run five CPU-heavy JavaScript loops in parallel on one main thread; use sandboxed processors or worker threads for CPU-heavy work.

## 7. Error handling and Redis outage
Throw an error in a processor when the operation failed; this enables queue failure/retry handling. Catch, log and intentionally decide whether some errors are permanent. Configure Redis connectivity, network timeouts, graceful shutdown and monitoring for production. Worker connection behavior differs from producer connection behavior; use version-supported options.

## 8. Job IDs, duplicates and idempotency
You can provide `jobId` for deduplication while a matching job still exists:
```js
await receiptQueue.add("send-receipt", { orderId }, {
  jobId: `receipt-${orderId}`,
});
```
**Important:** removed completed jobs no longer prevent re-adding that ID, and job-ID deduplication does not guarantee a side effect happens only once. Enforce DB/business idempotency (Lesson 200).

## 9. Queue vs QueueEvents
A worker listens to events for jobs it processes. `QueueEvents` provides queue-wide lifecycle notifications. Use monitoring without treating event streams as a guaranteed substitute for persisted job/business state.

## 10. Production checklist
Shared secured Redis; controlled job retention; bounded concurrency; transient retries with jitter/backoff; durable business data; monitoring/backlog alerts; payload validation; secrets outside job data; graceful shutdown; idempotency; transactional outbox when DB/queue atomicity matters.

## Practice
Queue a `generate-invoice` job using only `orderId`; add two workers and watch processing. Simulate a failure to observe retry attempts. Record completion in MongoDB.

**Interview answer:** BullMQ uses Redis to run asynchronous jobs with producers, workers, retries and scheduling. It decouples APIs from slow work, but requires idempotent processing and operational controls for reliability.
