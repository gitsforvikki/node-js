# Lesson 197 — Job Retries

## 1. What is a job retry?
A **retry** means attempting a failed job again. Imagine ShopHub's worker tries to email a receipt, but the email provider temporarily returns HTTP 503. Instead of losing the receipt, the queue can try again.

```text
waiting -> active -> provider fails
                   -> delay -> waiting -> active -> success
                                           |
                                           -> retry limit exhausted -> failed
```

A retry is **not** always correct. An invalid email address or bad input is usually a permanent failure; retrying it repeatedly wastes resources.

## 2. Transient vs permanent failures

| Failure | Typical action | Why |
| --- | --- | --- |
| Connection timeout | Retry with delay | Service may recover |
| HTTP 429 | Respect Retry-After and retry | Temporary quota limit |
| HTTP 503 | Retry with bounded backoff | Temporary unavailability |
| Invalid payload | Fail permanently | Repeating won't fix input |
| Permission denied | Investigate; usually fail | Configuration/security issue |
| Payment result unknown | Reconcile first, don't blindly charge again | Possible duplicate side effect |

## 3. BullMQ setup

Install `npm install bullmq`; start Redis locally. Add attempts when enqueueing:

```js
import { Queue } from "bullmq";

const emailQueue = new Queue("emails", {
  connection: { host: "127.0.0.1", port: 6379 },
});

await emailQueue.add("send-receipt", { orderId: "ord-123" }, {
  attempts: 4, // Includes original attempt
  backoff: { type: "fixed", delay: 2000 },
  removeOnComplete: { count: 1000 },
});
```

**Meaning:** BullMQ can run the processor at most 4 times for this job under the configured attempts policy; the delay between failed attempts is 2 seconds in this simple example.

## 4. Worker example

```js
import { Worker } from "bullmq";

const worker = new Worker("emails", async (job) => {
  const order = await Order.findById(job.data.orderId);
  if (!order) throw new Error("Missing order");
  await emailProvider.sendReceipt(order); // May throw on provider failure
}, { connection: { host: "127.0.0.1", port: 6379 } });

worker.on("failed", (job, error) => {
  console.error("Failed job", job?.id, error.message);
});
```

This worker assumes the database and email provider are initialized. The processor **must throw/reject** when a genuine failure occurs; catching and returning normally marks it successful.

## 5. Don't retry permanent errors

BullMQ provides `UnrecoverableError` for failures that should not be retried:

```js
import { UnrecoverableError } from "bullmq";

if (!order.customerEmail) {
  throw new UnrecoverableError("Customer has no email");
}
```

Avoid leaking customer data in error messages or logs.

## 6. Why retries can duplicate effects

```text
Worker sends receipt successfully
   -> crashes before marking job completed
   -> job is recovered and processed again
   -> another receipt may be sent
```

Queues generally provide **at-least-once-style processing**, not guaranteed exactly-once effects. Use business idempotency (Lesson 200), provider idempotency where supported, and explicit status tracking.

## 7. Dead-letter handling and investigation

After all attempts fail, retain enough metadata to investigate (with bounded retention). Depending on broker and design, move permanently failing messages to a **dead-letter queue** or record them in a failures table. BullMQ's failed jobs aren't automatically a separate DLQ.

Track failed count, oldest waiting age, retry attempts and repeat-failure categories. Don't retry poison messages forever.

## 8. Common mistakes

- Assuming `attempts: 3` means 3 retries *after* an initial attempt; it means 3 total attempts.
- Retrying validation and authentication failures.
- Retrying payments without reconciliation/idempotency.
- Swallowing a processor exception.
- Infinite retries during an outage, overwhelming a recovering provider.
- Removing failed job metadata before diagnosing the incident.

## 9. Practice
Create a worker that throws on its first two attempts and succeeds on the third. Log the job ID and `attemptsMade`, observe delays, and explain how to avoid duplicate receipts.

**Interview answer:** Job retries recover from transient failures using bounded attempts and delays. Distinguish permanent failures, monitor exhausted jobs, and make side effects idempotent because a job can execute more than once.
