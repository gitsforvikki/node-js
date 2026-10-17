# Lesson 200 — Idempotent Jobs

## 1. What does idempotent mean?
An operation is **idempotent** if performing it multiple times has the same intended effect as performing it once.

```text
Job: mark order ord-7 as CONFIRMED

1st execution -> status CONFIRMED
2nd execution -> status still CONFIRMED
3rd execution -> status still CONFIRMED
```

Compare a non-idempotent operation:
```js
await Wallet.updateOne({ _id: id }, { $inc: { balance: 100 } });
```
If run twice, it adds 200—not the intended 100.

## 2. Why queue jobs can execute twice
Worker completes an external side effect → crashes before acknowledging → queue recovers and retries. Message brokers don't generally guarantee exactly-once **business effects**.

## 3. Simple MongoDB status update
```js
await Order.updateOne(
  { _id: orderId, status: "CREATED" },
  { $set: { status: "CONFIRMED" } }
);
```
This conditional update avoids repeatedly transitioning the same order. Verify valid state transitions and handle concurrent state changes. It is not sufficient to protect an external charge/email by itself.

## 4. Idempotency key with a unique database index

Suppose the API receives `Idempotency-Key: checkout-abc`. Store it with the operation in a database collection indexed uniquely **per user/business operation**.

```js
const requestSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, required: true },
  key: { type: String, required: true },
  status: { type: String, enum: ["processing", "completed"], required: true },
  orderId: mongoose.Schema.Types.ObjectId,
});
requestSchema.index({ userId: 1, key: 1 }, { unique: true });
```

**Flow:**
1. Authenticate user and validate the key and payload.
2. Atomically reserve the key (unique index).
3. If already completed, return the saved result.
4. If in progress, return appropriate conflict/pending response.
5. Perform the business operation with transaction/provider idempotency as needed.
6. Save stable completion/result; recover stuck reservations after failures.

**Important:** a unique key alone does NOT make a payment and DB write atomic. Use the payment provider's idempotency key and reconciliation to resolve unknown outcomes.

## 5. BullMQ jobId vs business idempotency

```js
await queue.add("receipt", { orderId }, {
  jobId: `receipt-${orderId}`,
});
```

A custom `jobId` helps avoid duplicate jobs **while the original is retained** in that queue. If completed jobs are removed, the same ID may be enqueued again. A worker can still rerun after a crash. Therefore enforce business invariants in persistent storage.

## 6. Real email example: at-most-once vs at-least-once
If you mark “sent” **before** calling provider, a crash may prevent delivery. Marking it **after** can cause duplicates if provider succeeded but acknowledgement failed.

Decide desired semantics and use a provider-supported idempotency key when available, plus a durable delivery record. Some email systems cannot guarantee exactly one actual inbox delivery—be explicit about limitations.

## 7. Payment caution
Never retry a charge merely because the HTTP request timed out. The provider may have charged already. Preserve a stable idempotency key, query/reconcile transaction state, and enforce unique order/payment references.

## 8. Concurrency and atomicity
Two workers can receive duplicate work at nearly the same moment. “Check then write” is a race:
```js
if (!(await findExisting(key))) await charge(); // NOT SAFE
```
Use database unique constraints/atomic operations and external provider idempotency for irreversible side effects.

## 9. Practice
Design a `generate-invoice` job that may be delivered three times. Persist an invoice with a unique `orderId` and prove only one invoice record can exist. Then explain the separate problem of sending the PDF by email twice.

**Interview answer:** Idempotency makes duplicate executions safe. Use unique operation keys, atomic conditional updates, provider idempotency and recovery strategies. Queue deduplication alone is not enough for exactly-once effects.
