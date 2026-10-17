# Lesson 198 — Exponential Backoff

## 1. Why wait longer between retries?
If 10,000 workers retry immediately when a payment/email service goes down, they can overwhelm it even more. **Backoff** increases the delay before each new attempt.

```text
Attempt 1: fails
wait 1 second
Attempt 2: fails
wait 2 seconds
Attempt 3: fails
wait 4 seconds
Attempt 4: succeeds
```

## 2. Fixed vs exponential vs jitter

| Strategy | Delays | Tradeoff |
| --- | --- | --- |
| Immediate | 0, 0, 0 | May overload dependencies |
| Fixed | 1s, 1s, 1s | Predictable, can synchronize |
| Exponential | 1s, 2s, 4s | Gives provider increasing recovery time |
| Exponential + jitter | Randomized around growing delay | Reduces synchronized retry spikes |

Typical formula for retries:

```text
delay = min(maxDelay, baseDelay × 2^(retryNumber - 1))
```

Here `retryNumber=1` is the first retry, after the initial failure. Cap maximum delay and retry count.

## 3. BullMQ exponential backoff

```js
await emailQueue.add("send", { orderId: "ord-1" }, {
  attempts: 5,
  backoff: {
    type: "exponential",
    delay: 1000,
    jitter: 0.5,
  },
});
```

This configures exponential delays with randomized jitter between attempts in BullMQ versions supporting the `jitter` option. Verify behavior against your installed version. `attempts: 5` includes the first execution, allowing up to four retries.

## 4. Runnable retry helper (ordinary async function)

```js
import { setTimeout as sleep } from "node:timers/promises";

async function retryTransient(operation, maxAttempts = 4) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxAttempts || !isTransient(error)) throw error;

      const base = Math.min(8000, 500 * 2 ** (attempt - 1));
      const delay = Math.random() * base; // full jitter
      await sleep(delay);
    }
  }
}

function isTransient(error) {
  return error.status === 429 || error.status === 503 ||
    error.code === "ETIMEDOUT";
}
```

The demo assumes a structured error object. Real applications should adapt provider-specific errors, use abort signals/timeouts, honor a valid `Retry-After` header when relevant, and avoid exceeding an overall request/job deadline.

## 5. Execution flow
First call fails → classify transient → compute capped random delay → wait asynchronously → call again → return successful result or rethrow final failure.

Because `await sleep()` is nonblocking, **other Node.js requests can run** during the wait.

## 6. Retry storms and circuit breakers

Retries multiply traffic. Pair bounded retries with:
- concurrency limits and rate limits;
- jitter;
- circuit breakers that temporarily stop calls to unhealthy dependencies;
- queues/backpressure;
- monitoring and alerts.

**Circuit breaker** is not a replacement for retry; it prevents repeated attempts during sustained failure.

## 7. Idempotency matters

Never blindly retry an operation that might already have succeeded. Example: a payment API times out *after* charging the card. Use its supported idempotency key and reconciliation procedure before retrying.

## 8. Common mistakes
Using exponential backoff without a cap; retrying every 4xx response; forgetting jitter; ignoring Retry-After; sleeping synchronously; retrying inside both application and queue (multiplicative attempts).

## 9. Practice
Calculate four delays starting with 500 ms and doubling. Then simulate 20 simultaneous failed requests and compare fixed delay with full jitter.

**Interview answer:** Exponential backoff increases waiting time between retries to let services recover. Use a maximum delay, finite attempts, jitter and error classification; combine with idempotency and provider retry guidance.
