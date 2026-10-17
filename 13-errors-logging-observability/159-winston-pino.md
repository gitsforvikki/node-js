# Lesson 159 — Winston and Pino Logging in Node.js

## 1. Why use a logging library?

`console.log()` is useful while learning, but production applications benefit from structured output, levels, serializers, redaction, and integrations.

Two popular Node.js libraries are **Pino** and **Winston**.

| Feature | Pino | Winston |
| --- | --- | --- |
| Typical approach | Fast JSON structured logging | Flexible formats and transports |
| Best fit | High-throughput APIs and JSON logs | Apps needing varied log destinations |
| Levels | Supported | Supported |
| Redaction | Built-in redaction options | Requires careful format/filter setup |
| HTTP request logging | `pino-http` | `morgan` or custom middleware |

Neither tool automatically prevents sensitive data leakage: **you must configure what gets logged**.

## 2. Pino: recommended starting point for Express APIs

Install:

```bash
npm install pino pino-http
```

Create `src/logger.js`:

```js
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  redact: {
    paths: [
      "req.headers.authorization",
      "req.headers.cookie",
      "password",
      "token",
      "refreshToken",
    ],
    censor: "[REDACTED]",
  },
});
```

In `src/app.js`:

```js
import express from "express";
import pinoHttp from "pino-http";
import { logger } from "./logger.js";

const app = express();
app.use(pinoHttp({
  logger,
  // Do not log raw request bodies.
  autoLogging: true,
}));

app.get("/api/products", (req, res) => {
  req.log.info({ event: "products_listed" }, "Products requested");
  res.json({ products: [] });
});

app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);
  req.log.error({ err }, "Request failed");
  res.status(500).json({ message: "Internal server error" });
});
```

`req.log` is a request-aware logger. In production, configure trusted request IDs, error serializers, and safe request/response logging.

## 3. What does Pino output?

Example (illustrative):

```json
{"level":30,"time":1760000000000,"msg":"Products requested","event":"products_listed"}
```

Pino uses numeric level values by default (for example, info=30, error=50). JSON output is designed for log collection and filtering.

## 4. Pino child logger

A child logger attaches shared context:

```js
const orderLog = logger.child({ service: "checkout", orderId: "ord-42" });
orderLog.info({ event: "payment_started" }, "Payment started");
orderLog.error({ event: "payment_failed" }, "Payment failed");
```

Useful when multiple log lines belong to one order or job. Avoid putting secret or high-cardinality sensitive fields everywhere without a reason.

## 5. Winston: flexible formats and transports

Install:

```bash
npm install winston
```

```js
import winston from "winston";

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
  ],
});

logger.info("Server started", { service: "api" });
logger.error("Database unavailable", { code: "DB_DOWN" });
```

**Note:** Winston doesn't automatically redact every secret in arbitrary metadata. Add explicit sanitization/redaction before sending log data to transports.

## 6. Winston file transport (when needed)

```js
new winston.transports.File({
  filename: "logs/errors.log",
  level: "error",
});
```

This can help on a managed server, but containers usually favor stdout/stderr with centralized collection. If writing files, configure rotation, disk quotas, retention, and secure permissions.

## 7. How to choose?

Choose **Pino** when you want straightforward fast JSON logging for an Express API. Choose **Winston** when flexible transports/formats are central to your application.

Don't install both just because both are popular. Use one logging convention across the project.

## 8. Practical Express error logging

```js
app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);

  const isKnown = err instanceof AppError;
  const status = isKnown ? err.statusCode : 500;

  req.log.error({
    err,
    status,
    event: "http_request_failed",
  }, "API request failed");

  res.status(status).json({
    success: false,
    message: isKnown ? err.message : "Internal server error",
  });
});
```

This example assumes Pino HTTP middleware and `AppError` have already been set up. In production, distinguish expected 4xx responses from unexpected 5xx so routine invalid input doesn't generate noisy error alerts.

## 9. Logging safety checklist

- Never log passwords, JWTs, cookies, API keys, or full sensitive bodies.
- Redact nested sensitive fields and test the redaction.
- Log to stdout/stderr for most container deployments.
- Use request IDs to connect events.
- Avoid duplicate error logs at every layer.
- Keep retention and access permissions under control.
- Benchmark log volume in high-traffic services.

## 10. Practice and interview

1. Add Pino to an Express 5 API and inspect JSON output.
2. Attach a child logger to an order operation.
3. Configure Pino redaction and verify Authorization values do not appear.
4. Implement the same info/error logs with Winston.
5. Explain when you would choose Pino vs Winston.

**Interview answer:** Pino and Winston are production logging libraries. Pino focuses on fast structured JSON logs; Winston offers flexible formatting and transports. Both need sensible levels, secure metadata handling, and integration with request IDs and centralized log collection.
