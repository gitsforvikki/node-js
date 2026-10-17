# Lesson 153 — Custom Error Classes

## 1. Why create your own error class?

JavaScript has a built-in `Error`:

```js
throw new Error("User not found");
```

But an API also needs a **status code**, stable machine-readable **error code**, and a safe public message. A custom class keeps them together.

Think of an error object like a package:

```text
AppError
 ├─ message: "User not found"
 ├─ statusCode: 404
 ├─ code: "USER_NOT_FOUND"
 ├─ isOperational: true
 └─ stack: useful debugging location
```

## 2. Build AppError

```js
export class AppError extends Error {
  constructor(message, statusCode = 500, code = "INTERNAL_ERROR", options = {}) {
    super(message, { cause: options.cause });
    this.name = "AppError";
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
    Error.captureStackTrace?.(this, this.constructor);
  }
}
```

`extends Error` provides normal Error behavior; `super(message)` sets the message; `this` stores extra properties. The optional `cause` keeps a link to a lower-level error **for internal diagnostics**.

## 3. Use in a controller (Express 5)

```js
app.get("/products/:id", async (req, res) => {
  const product = await Product.findById(req.params.id);
  if (!product) {
    throw new AppError("Product not found", 404, "PRODUCT_NOT_FOUND");
  }
  res.json(product);
});
```

Execution:

```text
Product missing -> throw AppError
   -> Express 5 catches rejected handler Promise
   -> global error middleware
   -> HTTP 404 with PRODUCT_NOT_FOUND
```

## 4. Error subclass examples

Use subclasses if they genuinely improve readability:

```js
export class NotFoundError extends AppError {
  constructor(resource = "Resource") {
    super(`${resource} not found`, 404, "NOT_FOUND");
  }
}

export class ForbiddenError extends AppError {
  constructor() {
    super("Forbidden", 403, "FORBIDDEN");
  }
}
```

```js
if (!user) throw new NotFoundError("User");
if (order.userId !== req.user.id) throw new ForbiddenError();
```

Avoid creating dozens of subclasses where a simple `AppError` is clearer.

## 5. Catch a lower-level error and preserve cause

```js
try {
  await emailProvider.send(message);
} catch (cause) {
  throw new AppError(
    "Email service temporarily unavailable",
    503,
    "EMAIL_PROVIDER_UNAVAILABLE",
    { cause }
  );
}
```

The API sees a safe general explanation, while internal logging can record the underlying `cause`. Do **not** expose the internal cause in the HTTP response.

## 6. Expected errors vs code bugs

A custom class does not magically make a bug safe. Only throw `AppError` when the message and status are intentionally chosen.

```js
// BAD: accidentally tells client private details
throw new AppError(databaseError.message, 400, "DB_ERROR");

// BETTER: log privately, show a safe known message
throw new AppError("Unable to complete request", 503, "SERVICE_UNAVAILABLE", {
  cause: databaseError,
});
```

The controller should not convert **every** unexpected bug into an ordinary 400 response.

## 7. How to recognize a custom error

```js
if (err instanceof AppError) {
  console.log(err.statusCode, err.code);
}
```

`instanceof` detects objects made from the class (and subclasses). If errors cross worker/process boundaries or JSON serialization, they may lose prototype identity; normalize them at the boundary.

## 8. Standard API error response

```js
app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);
  const known = err instanceof AppError;
  const status = known ? err.statusCode : 500;

  res.status(status).json({
    success: false,
    code: known ? err.code : "INTERNAL_ERROR",
    message: known ? err.message : "Internal server error",
  });
});
```

Keep HTTP status between 400 and 599 (validate it in a larger app). Don't confuse the HTTP code `404` with business code `PRODUCT_NOT_FOUND`: the former is general, the latter is application-specific.

## 9. Where should errors be thrown?

- **Middleware:** authentication and validation failures.
- **Services:** business-rule failures such as insufficient balance.
- **Controllers:** orchestration and missing-resource responses.
- **Error handler:** turns errors into consistent responses.

Prefer predictable, stable error codes so the frontend can handle cases without parsing message text.

## 10. Common mistakes

1. Forgetting `super(message)` in a class extending `Error`.
2. Returning `200` for failures.
3. Exposing `err.stack` or `err.cause`.
4. Wrapping every bug as `AppError` and hiding programmer defects.
5. Using inconsistent codes across endpoints.
6. Making `cause` or raw third-party errors public.

## 11. Practice and interview

1. Create `ValidationError` (400) and `NotFoundError` (404).
2. Throw them from two routes and check the centralized handler response.
3. Wrap an email-service failure while preserving internal `cause`.
4. Explain `Error` vs `AppError` vs `instanceof`.

**Interview answer:** Custom error classes extend JavaScript Error to add HTTP status, stable codes, and internal metadata. Controllers/services throw them; centralized middleware converts known errors to safe responses and hides internal details for unexpected failures.
