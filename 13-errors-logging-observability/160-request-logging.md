# Lesson 160 — Request Logging in Express 5

## 1. What is request logging?

**Request logging** records useful details about an HTTP request and its result: method, route, status code, duration, and request ID.

Imagine a user says, “My checkout API is slow.” A request log can show **which endpoint**, **how long it took**, and **whether it failed**.

```text
Client -> Express middleware (start timer)
       -> authentication -> controller -> database
       -> response sent
       -> middleware logs method, route, status, duration
```

## 2. Request logs vs application logs

- **Request log:** `GET /api/products -> 200 in 34 ms`.
- **Application log:** `Inventory cache miss for product 42`.
- **Error log:** `Payment provider timed out`.

They complement one another. Do not log every request body or every database result.

## 3. Build simple request-logging middleware

```js
import express from "express";
import { performance } from "node:perf_hooks";

const app = express();

app.use((req, res, next) => {
  const start = performance.now();

  res.on("finish", () => {
    const durationMs = Math.round(performance.now() - start);
    console.log(JSON.stringify({
      event: "http_request",
      method: req.method,
      path: req.route?.path ?? req.path,
      statusCode: res.statusCode,
      durationMs,
    }));
  });

  next();
});

app.get("/api/products", (req, res) => {
  res.json({ products: [] });
});

app.listen(3000);
```

**Why `finish`?** It fires when Node has handed off the response data to the underlying system, not necessarily when the client has received it. `close` can indicate a prematurely closed response.

**Why `performance.now()`?** It measures elapsed time with a monotonic clock, avoiding wall-clock adjustments.

## 4. Important middleware execution detail

The logging middleware runs **before** the controller and registers a listener. When the response finishes, the listener records its result.

```text
Request arrives
  -> logging middleware records start time
  -> next() calls route
  -> route sends JSON
  -> finish event
  -> logger calculates duration
```

The route template `req.route?.path` may be unavailable for unmatched routes or middleware responses, and it does not include router mount paths. For production metrics/logging, prefer a normalized route label (e.g. `/api/products/:id`) rather than raw URLs with IDs or query strings.

## 5. Handle aborted requests too

A connection may close before the response finishes. To avoid double logging, use one guarded function:

```js
app.use((req, res, next) => {
  const start = performance.now();
  let recorded = false;

  function record(outcome) {
    if (recorded) return;
    recorded = true;
    console.log(JSON.stringify({
      event: "http_request",
      method: req.method,
      outcome,
      statusCode: res.statusCode,
      durationMs: Math.round(performance.now() - start),
    }));
  }

  res.once("finish", () => record("completed"));
  res.once("close", () => record(res.writableFinished ? "completed" : "aborted"));
  next();
});
```

The `close` event alone doesn't always mean a failed request; check whether the response completed.

## 6. Production logging with Pino HTTP

Install:

```bash
npm install pino pino-http
```

```js
import pino from "pino";
import pinoHttp from "pino-http";

const logger = pino({ level: process.env.LOG_LEVEL ?? "info" });

app.use(pinoHttp({
  logger,
  autoLogging: {
    ignore: (req) => req.url === "/health/live",
  },
}));

app.get("/api/orders", (req, res) => {
  req.log.info({ event: "orders_requested" }, "Fetching orders");
  res.json({ orders: [] });
});
```

`pino-http` generates request-completion logs and makes `req.log` available. Configure safe serializers/redaction, request IDs, and logging levels. Don't combine this with another automatic request logger unless duplicate logs are intentional.

## 7. What should be recorded?

Useful: method, normalized route, HTTP status, elapsed milliseconds, request ID, service, and error category. Sometimes include a trusted user ID if justified.

Avoid: raw `Authorization` headers, cookies, passwords, reset tokens, full bodies, and query strings containing private data.

## 8. Request logging vs tracing

A request log tells you that `POST /checkout` took 850 ms. A trace can show **where** the time was spent: authentication 5 ms, DB 70 ms, payment provider 750 ms.

Request logs are the starting point; tracing adds cross-service timing.

## 9. Common mistakes

1. Logging full URLs containing tokens or personal information.
2. Recording only successful requests and missing aborted connections.
3. Logging the same request twice through multiple middlewares.
4. Using raw unique URLs as metric labels (high cardinality).
5. Assuming `finish` guarantees the client received the bytes.
6. Treating all 4xx responses as critical server failures.

## 10. Practice and interview

1. Add the simple logger and call `GET /api/products`.
2. Make a route wait 300 ms and inspect `durationMs`.
3. Compare `finish` and `close`.
4. Replace `console.log` with Pino HTTP and explain the benefits.

**Interview answer:** Request logging records HTTP request metadata and outcome, especially method, route, status, duration, and request ID. It helps diagnose latency and failures. Use structured logging, avoid sensitive data, and account for aborted requests.
