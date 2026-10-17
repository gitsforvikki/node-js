# Lesson 158 — Structured Logging

## 1. What is logging?

A **log** is a record of what happened inside an application: request received, order created, database unavailable, job failed, etc.

A plain log is human-readable:

```js
console.log("Order failed for user 42");
```

But searching thousands of plain-text lines is difficult.

## 2. What is structured logging?

**Structured logging** stores logs as consistent fields, often JSON:

```json
{
  "level": "error",
  "event": "order_payment_failed",
  "requestId": "req-123",
  "orderId": "ord-42",
  "message": "Payment provider timed out"
}
```

You can now filter by `level`, `requestId`, `orderId`, or `event` in your log platform.

## 3. Real-world example

Imagine 1,000 users are checking out simultaneously. One user reports a failed order. Without structured logs, you may search through hundreds of strings. With a request ID, you can trace:

```text
requestId=req-123
   -> request started
   -> order validated
   -> payment call timed out
   -> HTTP 503 returned
```

## 4. Logging levels

| Level | Meaning | Example |
| --- | --- | --- |
| fatal | Process cannot safely continue | Unrecoverable startup failure |
| error | Operation failed | DB query error |
| warn | Something unusual, app still works | Retry triggered |
| info | Important normal event | Order created |
| debug | Development diagnostics | Cache decision |
| trace | Very detailed diagnostics | Internal step-by-step details |

Don't log every normal request as an error. Choose levels consistently.

## 5. Basic structured logger (learning example)

```js
function log(level, event, details = {}) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    event,
    ...details,
  }));
}

log("info", "order_created", {
  orderId: "ord-42",
  requestId: "req-123",
});
```

Output:

```json
{"timestamp":"2026-01-01T10:00:00.000Z","level":"info","event":"order_created","orderId":"ord-42","requestId":"req-123"}
```

This is an educational implementation. In production, use a maintained logger such as **Pino or Winston** (Lesson 159), which handles formatting, levels, performance, and integrations more reliably.

## 6. What should you log?

Useful fields:
- timestamp, level, event/message
- service name and environment
- request/correlation ID
- HTTP method, route template, status, duration
- trusted user ID **only when necessary and permitted**
- relevant entity ID (order ID, job ID)
- error name, code, and stack for internal diagnostics

Prefer **route templates** like `/orders/:id` over full URLs containing private query parameters.

## 7. Never log secrets or private content

Avoid logging:
- passwords and password hashes
- Authorization headers, access/refresh tokens, session cookies
- OTPs, reset links, API keys
- complete payment details
- raw request bodies containing personal data

Redaction must happen **before** data reaches storage. Logging libraries can redact known fields, but review nested objects and custom serializers.

## 8. Log errors correctly

```js
try {
  await chargePayment();
} catch (err) {
  logger.error({
    err,
    event: "payment_failed",
    orderId,
    requestId,
  }, "Payment failed");
  throw err;
}
```

For Pino, `err` is a recognized error serialization field. Log meaningful context and propagate the error rather than silently swallowing it.

**Avoid duplicate logging:** If every service and controller logs the same error at `error` level, one failure may create ten alerts. Decide which layer owns the final error log.

## 9. Logs vs metrics vs traces

- **Logs:** What happened? ("payment failed")
- **Metrics:** How often/how much? (failure rate 3%)
- **Traces:** Where did time go? (API → DB → payment provider)

They complement each other. Request IDs and trace IDs help connect events.

## 10. Performance and production

Structured logs are usually shipped from **stdout/stderr** to your deployment platform or log collector. Avoid writing an unbounded local log file in containers.

Logging too much can increase CPU, storage cost, and privacy risk. Set sensible levels, retention, access control, and sampling where appropriate.

## 11. Practice

1. Convert three `console.log` strings into JSON objects.
2. Add `requestId` to all logs for a request.
3. Log a failed order without including payment credentials.
4. Explain how logs differ from metrics and traces.

**Interview answer:** Structured logging records events with consistent machine-readable fields (usually JSON). It improves searching, filtering, debugging, and alerting. Include request IDs and useful context, use appropriate levels, and redact secrets.
