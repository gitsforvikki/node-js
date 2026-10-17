# Lesson 193 — Message Queues

## 1. Message queue in simple words
A **message queue** stores work/messages so the sender and receiver don't have to run at the same moment.

Think of a parcel pickup counter: a customer drops a package, gets a receipt, and leaves; delivery workers process packages later.

```text
Order API (producer) -> Queue -> Email worker (consumer)
                              -> Invoice worker (another queue)
```

Without a queue, an email-provider outage might delay every checkout. With one, the order API can continue if enqueueing succeeds and workers catch up later.

## 2. Four basic concepts
- **Producer:** creates messages/jobs.
- **Queue/broker:** holds them until consumed (subject to persistence/retention).
- **Consumer/worker:** receives and processes them.
- **Acknowledgment (ack):** signals that work was handled so the broker may remove or mark it complete.

## 3. Queue vs Pub/Sub
| Queue | Pub/Sub |
| --- | --- |
| A work item normally goes to one consumer in a worker group | Every active subscriber to a channel can receive the broadcast |
| Useful for tasks | Useful for live notifications/updates |
| Can retain work until worker available | Redis Pub/Sub does **not** retain messages for offline subscribers |
| Ack/retry depends on broker | Redis Pub/Sub has no per-message consumer ack |

For durable event fan-out, look at Redis Streams, Kafka, RabbitMQ or other systems with explicit delivery/retention semantics—do not assume basic Pub/Sub is a job queue.

## 4. Lifecycle and acknowledgment
```text
Producer publishes job
  -> waiting
  -> worker receives
  -> processing
  -> ack/complete or failure
  -> retry, dead-letter handling, or permanent failure
```
If a worker crashes **before acknowledgment**, many queue systems redeliver. Processing might therefore happen **more than once**.

## 5. Delivery guarantees
- **At-most-once:** message can be lost; not intentionally redelivered.
- **At-least-once:** message may be redelivered; handler must tolerate duplicates.
- **Exactly-once effect:** a business goal requiring deduplication/transactions/idempotency; not guaranteed merely by installing a queue.

Also distinguish **ordering**: concurrent consumers can finish jobs in a different order from enqueueing.

## 6. Practical design: order and email
Imagine:
```js
await orders.create({ customerId, status: "PAID" });
await emailQueue.add("receipt", { orderId });
```
What if the process crashes between these lines? Order is saved but email job is missing. For important events, use **outbox**: write order + outbox record atomically, then a publisher retries enqueueing until confirmed.

## 7. Queue reliability vocabulary
**Backpressure:** when producers enqueue faster than workers can process, backlog grows; apply rate limits/quotas and autoscale carefully. **Dead-letter queue:** stores jobs that repeatedly fail for investigation. **Visibility timeout/lock:** broker-specific period when a worker owns a job. **Poison message:** message that reliably fails due to bad content; retrying it forever is harmful.

## 8. Which broker?
- **BullMQ + Redis:** convenient for Node.js background jobs and delayed work.
- **RabbitMQ:** flexible routing, acknowledgments and durable messaging options.
- **Kafka:** high-volume append-only event streams with consumer offsets.
Selection depends on delivery semantics, scale, operations, replay and ordering requirements.

## 9. Security + operations
Validate message schema in worker; assume messages can be malformed/outdated. Restrict broker access; avoid personal secrets in payloads; measure backlog, job age, error rate, attempt counts and worker capacity.

## Practice
Draw a pipeline for orders → email and invoices. Mark which operations may fail, where retry happens, and how the system avoids sending two receipts.

**Interview answer:** Queues decouple producers from consumers and smooth workload spikes. Consumer acknowledgment, retention, retries, dead-letter handling, ordering, and idempotency determine whether processing is reliable.
