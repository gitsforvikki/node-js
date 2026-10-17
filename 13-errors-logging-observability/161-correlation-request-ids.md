# Lesson 161 — Correlation IDs and Request IDs

## 1. What problem do request IDs solve?

Many users call your API simultaneously. Logs from their requests become mixed together.

A **request ID** is a unique identifier attached to one incoming HTTP request. It lets you find all log entries belonging to that request.

```text
Request A (req-a1) -> auth -> orders -> response
Request B (req-b2) -> auth -> products -> response

Logs:
req-a1: started
req-b2: started
req-a1: DB query
req-b2: completed
req-a1: completed
```

Search `req-a1` to follow only Request A.

## 2. Request ID vs correlation ID vs trace ID

- **Request ID:** identifies a particular HTTP request.
- **Correlation ID:** connects related work across requests/services (sometimes reused across multiple hops).
- **Trace ID:** identifies an entire distributed trace; **span IDs** identify individual operations within it.

Teams sometimes use these terms loosely, but they are not always interchangeable. For distributed tracing, prefer standard W3C Trace Context rather than inventing a custom tracing protocol.

## 3. Express 5 example: generate a trusted ID

```js
import { randomUUID } from "node:crypto";

app.use((req, res, next) => {
  req.requestId = randomUUID();
  res.setHeader("X-Request-ID", req.requestId);
  next();
});

app.get("/api/orders", async (req, res) => {
  console.log(JSON.stringify({
    requestId: req.requestId,
    event: "orders_fetch_started",
  }));
  res.json({ requestId: req.requestId, orders: [] });
});
```

**Why generate it server-side?** A client-supplied `X-Request-ID` can contain misleading text or be reused deliberately. You can accept upstream IDs **only from trusted proxies/services** after validation. Never use a request ID as authentication or authorization.

## 4. Request ID propagation to services

```js
async function getOrders(userId, requestId) {
  logger.info({ requestId, userId, event: "orders_query" });
  return Order.find({ userId });
}

app.get("/api/orders", requireAuth, async (req, res) => {
  const orders = await getOrders(req.user.id, req.requestId);
  res.json({ orders });
});
```

Passing IDs explicitly is simple and easy to understand. It becomes repetitive across many service functions.

## 5. AsyncLocalStorage: context without passing every parameter

Node.js provides `AsyncLocalStorage` in `node:async_hooks` to carry context through many asynchronous operations.

```js
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

export const requestContext = new AsyncLocalStorage();

app.use((req, res, next) => {
  const requestId = randomUUID();
  req.requestId = requestId;
  res.setHeader("X-Request-ID", requestId);
  requestContext.run({ requestId }, () => next());
});

function logEvent(event) {
  const requestId = requestContext.getStore()?.requestId;
  console.log(JSON.stringify({ requestId, event }));
}

app.get("/api/example", async (req, res) => {
  logEvent("started");
  await Promise.resolve();
  logEvent("after_await");
  res.json({ ok: true });
});
```

Expected logs have the **same requestId** before and after `await`. Most standard Promise/timer APIs preserve context; some custom callback integrations may need special handling.

## 6. Integrate with Pino

```js
import pino from "pino";

const logger = pino();

function getRequestLogger() {
  const requestId = requestContext.getStore()?.requestId;
  return logger.child({ requestId });
}

getRequestLogger().info({ event: "payment_started" }, "Payment started");
```

In a real application, create/reuse request-aware loggers rather than unnecessarily creating child loggers for every statement. Pino HTTP also supports request ID configuration.

## 7. Calling another service

Suppose your API calls a payment service:

```js
const response = await fetch("https://payments.example/api/charge", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-Request-ID": req.requestId,
  },
  body: JSON.stringify(payload),
});
```

This is illustrative. Forward correlation headers only to **trusted services**. For actual distributed tracing, propagate `traceparent` using OpenTelemetry/W3C Trace Context. **A request ID is not an idempotency key**: it does not prevent duplicate payments.

## 8. Security and privacy considerations

- Generate unpredictable IDs or validate trusted upstream IDs.
- Never use IDs as proof of identity.
- Do not embed secrets, emails, or phone numbers in IDs.
- Avoid reflecting arbitrary attacker-controlled IDs into logs.
- Use log retention and access controls.

## 9. Common mistakes

1. Generating a different ID in every controller.
2. Forgetting to attach ID to error logs.
3. Assuming request IDs automatically trace external services.
4. Using a request ID as an authorization or payment-idempotency token.
5. Trusting arbitrary client-provided correlation headers.
6. Losing context in custom async callback systems.

## 10. Practice and interview

1. Send two API requests and confirm they receive different IDs.
2. Use `AsyncLocalStorage` to log the same ID before/after an `await`.
3. Attach the ID to a centralized error response/log.
4. Explain the difference between request ID, trace ID, and idempotency key.

**Interview answer:** Request IDs link all logs for one HTTP request. Generate them at the boundary, propagate them through trusted services, and attach them to structured logs. AsyncLocalStorage can preserve request context across asynchronous operations.
