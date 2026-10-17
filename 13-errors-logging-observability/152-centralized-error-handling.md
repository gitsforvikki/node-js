# Lesson 152 — Centralized Error Handling (Express 5)

## 1. What problem does it solve?

Without centralized handling, every controller repeats response logic:

```js
// Repeated in dozens of controllers — hard to keep consistent
try {
  const user = await User.findById(id);
  res.json(user);
} catch (err) {
  res.status(500).json({ message: "Something went wrong" });
}
```

With a **central error handler**, controllers can focus on business logic. Your API has one consistent format for error responses and logging.

## 2. A minimal Express 5 example

```js
import express from "express";
const app = express();

app.get("/demo", async (req, res) => {
  throw new Error("Unexpected DB result");
});

app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ success: false, message: "Internal server error" });
});

app.listen(3000);
```

Express recognizes error middleware by its **four arguments**: `(err, req, res, next)`.

## 3. How does execution work?

```text
Request -> route -> throw Error
                       |
                  Express 5
                       |
                       v
            error middleware (err, req, res, next)
                       |
                 send safe response
```

The error middleware belongs **after your routes**. Normal middleware uses `(req, res, next)`; error middleware uses `(err, req, res, next)`.

## 4. AppError + central handler

```js
export class AppError extends Error {
  constructor(message, statusCode = 500, code = "INTERNAL_ERROR") {
    super(message);
    this.name = "AppError";
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
    Error.captureStackTrace?.(this, this.constructor);
  }
}
```

Controller (Express 5):

```js
app.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError("User not found", 404, "USER_NOT_FOUND");
  res.json({ success: true, data: user });
});
```

Central handler:

```js
app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);

  console.error({ message: err.message, stack: err.stack });

  const known = err instanceof AppError;
  const status = known && err.statusCode >= 400 && err.statusCode < 600
    ? err.statusCode : 500;

  res.status(status).json({
    success: false,
    code: known ? err.code : "INTERNAL_ERROR",
    message: known ? err.message : "Internal server error",
  });
});
```

**Important:** `AppError` should be used only for intentionally safe, public-facing messages; don't wrap raw database error text inside it.

## 5. Why Express 5 simplifies async controllers

```js
// Express 5: rejection automatically reaches error middleware
app.get("/items", async (req, res) => {
  const items = await Item.find();
  res.json(items);
});
```

You generally do **not** need a `try/catch` simply to call `next(err)` when the error comes from a returned Promise. Express 4 was different; see Lesson 154.

## 6. What about validation and database errors?

Prefer preventing invalid database inputs with validation:

```js
import mongoose from "mongoose";

app.get("/users/:id", async (req, res) => {
  if (!mongoose.isObjectIdOrHexString(req.params.id)) {
    throw new AppError("Invalid user ID", 400, "INVALID_ID");
  }
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError("User not found", 404, "USER_NOT_FOUND");
  res.json({ id: user.id, name: user.name });
});
```

A MongoDB duplicate-key failure (often `code === 11000`) may map to a **409 conflict**, but only map recognized cases; never expose database internals.

## 7. Handle 404 routes separately

No matching route is not necessarily an error thrown by Express:

```js
app.use((req, res, next) => {
  next(new AppError("Route not found", 404, "ROUTE_NOT_FOUND"));
});

// Error handler MUST come after this
app.use(errorHandler);
```

A missing database record is handled in the controller; an unmatched API path can be handled by a final not-found middleware.

## 8. Headers already sent

Once the response begins streaming, you cannot safely send a new JSON error response. If `res.headersSent` is true, delegate via `next(err)` so Express can close/finalize appropriately.

Do not try `res.status(500).json(...)` after sending headers; this often causes `ERR_HTTP_HEADERS_SENT`.

## 9. Avoid catching and swallowing errors

```js
// BAD — failure may leave client hanging
app.get("/bad", async (req, res) => {
  try {
    await doWork();
  } catch (err) {
    console.error(err); // no response, no rethrow!
  }
});

// GOOD — let Express 5 handle it
app.get("/good", async (req, res) => {
  await doWork();
  res.sendStatus(204);
});
```

Use `try/catch` when you can **recover, compensate, transform, or add context**—not just to repeat error forwarding.

## 10. Recommended project organization

```text
src/
  app.js
  routes/user.routes.js
  controllers/user.controller.js
  errors/AppError.js
  middleware/notFound.js
  middleware/errorHandler.js
```

Order in `app.js`: security + parsers → routes → notFound → errorHandler.

## 11. Testing

- Wrong object ID → 400.
- Missing record → 404.
- Unexpected exception → 500 with safe message.
- Invalid request body → 400/422 according to API design.
- Unmatched path → 404.
- Error after partial streaming response → no second JSON response.

## 12. Practice and interview

Write a controller that throws `AppError` for missing data and let the global handler return consistent JSON.

**Interview answer:** Centralized error middleware separates error responses from business logic, standardizes status codes/logging, and prevents leaking sensitive internals. Express error handlers take four parameters and are registered after routes; Express 5 automatically propagates rejections from returned async handlers.
