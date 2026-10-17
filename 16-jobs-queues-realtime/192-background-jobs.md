# Lesson 192 — Background Jobs

## 1. What problem does a background job solve?
An HTTP client expects a quick response. But sending 1,000 emails, generating a PDF or processing a video may take seconds or minutes. A **background job** moves that work outside the request/response path.

**Real example — ShopHub order confirmation:** checkout creates the order, then an independent worker sends confirmation email. The customer shouldn't wait for the email provider.

```text
Browser -> POST /orders -> API validates + saves order
                              |
                              +--> adds "send-confirmation" job to queue
                              |
                         HTTP 201 response
                                       |
                                 Worker receives job
                                       |
                                  Sends email
```

## 2. Why simply calling an async function is not enough
```js
app.post("/orders", async (req, res) => {
  const order = await Order.create(req.body);
  sendEmail(order); // BAD: detached Promise; failure may be unhandled
  res.status(201).json(order);
});
```
Even with `void sendEmail(order).catch(logger.error)`, a process crash/redeployment loses work in progress. A queue can retain and retry work **according to its durability configuration**.

## 3. A real BullMQ producer + worker
Prerequisites: Node.js with ESM, Redis running locally (`docker run -d --name redis-jobs -p 6379:6379 redis:7`); install `npm i express bullmq ioredis`.

**queue.js**
```js
import { Queue } from "bullmq";
export const emailQueue = new Queue("emails", {
  connection: { host: "127.0.0.1", port: 6379 },
});
```
**app.js**
```js
import express from "express";
import { emailQueue } from "./queue.js";
const app = express();
app.use(express.json());

app.post("/orders/:id/confirmation", async (req, res) => {
  // Real app: authenticate, authorize ownership, validate ID,
  // and ensure order already exists in DB.
  const job = await emailQueue.add("confirmation", {
    orderId: req.params.id,
  });
  res.status(202).json({ jobId: job.id, status: "queued" });
});
app.listen(3000);
```
**worker.js** (run as a separate process):
```js
import { Worker } from "bullmq";
const worker = new Worker("emails", async (job) => {
  const { orderId } = job.data;
  console.log("Send confirmation for order:", orderId);
  // Look up order in DB, render template, call email provider.
  // Throw if sending fails so the queue can retry.
}, { connection: { host: "127.0.0.1", port: 6379 } });

worker.on("failed", (job, err) => {
  console.error("Email job failed:", job?.id, err.message);
});
```

Run API and worker in **different terminals**. POST to the example route and watch worker output. The example *logs* rather than actually sending mail.

## 4. Background job lifecycle
A job can be waiting → active → completed, or failed → retried/failed permanently. BullMQ also supports delayed jobs and repeat scheduling.

**Important:** If saving the order succeeds but queue insertion fails, the system can become inconsistent. For critical workflows, consider the **transactional outbox pattern**: save the order and an outbox event in the same DB transaction, then asynchronously publish it to the queue.

## 5. When to return 201 vs 202
- `201 Created`: order itself has been created.
- `202 Accepted`: work was accepted for asynchronous processing, but **not yet completed**.
Track job state using a database status or an authorized status endpoint. Do not promise a finished email when you've only queued it.

## 6. Production considerations
- Retry recoverable errors with bounded backoff.
- Make handlers **idempotent**; duplicate execution can occur.
- Store minimum job data: IDs rather than huge objects/secrets.
- Cap workers/concurrency and monitor queue wait time, failures, lag.
- Use shared Redis with safe credentials, persistence/failover appropriate to workload.
- Gracefully close workers/queues on SIGTERM.
- Never run user-provided JavaScript as a job.

## 7. Common mistakes and practice
Mistakes: using `setTimeout()` as a reliable job queue; reporting success before saving critical data; leaving rejected Promises unhandled; assuming exactly-once execution; putting raw access tokens in job payloads.

Practice: Queue `generate-invoice` using `orderId`; worker should fetch the order from MongoDB, update invoice status, and handle missing orders safely.

**Interview answer:** Background jobs separate slow/nonessential work from HTTP requests. Producers enqueue durable work, consumers process it separately, and reliability requires retry policy, idempotency, monitoring and careful consistency between the database and queue.
