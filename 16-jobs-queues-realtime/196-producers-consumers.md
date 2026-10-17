# Lesson 196 — Producers and Consumers

## 1. Two responsibilities
A **producer** creates a message/job; a **consumer** processes it.

For your developer-chat style application:
- Producer: API saves message and queues a notification.
- Consumer: worker fetches recipient preferences and sends an email/push alert.

```text
POST /messages -> DB write -> notification producer -> Redis queue
                                                  |
                                            notification consumer
                                                  |
                                           email/push provider
```

## 2. Why keep them separate?
Producers are optimized for fast HTTP responses. Consumers can scale independently, retry and handle provider outages without keeping user requests open.

**Producer**:
```js
import { Queue } from "bullmq";
const notifications = new Queue("notifications", {
  connection: { host: "127.0.0.1", port: 6379 },
});

const job = await notifications.add("new-message", {
  messageId: "msg-123",
});
console.log("queued:", job.id);
```

**Consumer**:
```js
import { Worker } from "bullmq";
const worker = new Worker("notifications", async (job) => {
  const { messageId } = job.data;
  const message = await Message.findById(messageId);
  if (!message) return; // define whether missing entity is permanent
  await sendNotification(message); // throws on transient failure
}, { connection: { host: "127.0.0.1", port: 6379 }, concurrency: 5 });
worker.on("failed", (job, err) => console.error(job?.id, err));
```
These code snippets illustrate the roles; the consumer assumes you imported a real `Message` model and `sendNotification()` function and connected MongoDB at startup.

## 3. Jobs vs events
- **Command/job:** “send receipt for order 123” → someone should do this task.
- **Event:** “order 123 was paid” → multiple interested systems may react.

Different queue technologies support different routing/fan-out semantics. BullMQ jobs generally go to **one worker instance per execution**, not to every worker.

## 4. Scaling workers
One worker gets overwhelmed → add more worker instances. But increasing concurrency can overwhelm MongoDB or email providers. Bound concurrency, rate-limit external calls, monitor queue depth and oldest waiting job age.

## 5. Multiple consumer groups
In a durable stream/broker, independent consumer groups may each receive and process the same logical event (e.g., search indexing and analytics), while members **within** a group share work. Basic work queues and Redis Pub/Sub differ; don't assume every queue provides this model.

## 6. Failure scenario
Worker sends email → crashes before acknowledging → job gets redelivered → email may send again. Design with idempotency keys/provider dedupe where possible, and durable delivery status in your DB.

## 7. Schema versioning
Worker deployment may be older than the producer. Include a known `type` and `version` when evolving job payloads; validate input and keep backward-compatible processing during rolling deployments.

## 8. Security
Do not trust queue payloads blindly. Validate IDs and ownership at the appropriate boundary; use least-privilege Redis credentials and avoid placing tokens or passwords in messages.

## Practice
Draw two producers (signup + order API) and two separate consumers (welcome-email + receipts). State what happens when Redis is unavailable, the worker crashes, or the email provider is down.

**Interview answer:** Producers enqueue work and consumers execute it. The separation smooths traffic spikes and enables independent scaling, but correctness requires acknowledgment, retry control, schema validation and idempotency.
