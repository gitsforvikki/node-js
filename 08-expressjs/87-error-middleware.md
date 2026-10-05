# Lesson 87 — Error-handling Middleware

## Why this lesson is critical

A production Express application should not handle errors differently in every route.

Centralized error handling gives you:

- consistent responses
- secure error messages
- centralized logging
- less duplicated try/catch logic
- easier monitoring
- predictable API behavior

---

## 1. Error middleware signature

Express identifies error middleware by four arguments:

```js
function errorHandler(
  err,
  req,
  res,
  next
) {
  // ...
}
```

Important:

```text
(err, req, res, next)
```

not:

```text
(req, res, next)
```

---

## 2. Register it last

Typical app:

```js
app.use(apiRouter);

app.use(notFoundHandler);

app.use(errorHandler);
```

Error handling should come after routes/middleware that may generate errors.

---

## 3. Passing errors

Synchronous middleware:

```js
function middleware(
  req,
  res,
  next
) {
  try {
    riskyOperation();
    next();
  } catch (error) {
    next(error);
  }
}
```

Async:

```js
async function controller(
  req,
  res,
  next
) {
  try {
    const user =
      await service();

    res.json(user);
  } catch (error) {
    next(error);
  }
}
```

---

## 4. Central handler

```js
function errorHandler(
  err,
  req,
  res,
  next
) {
  console.error(err);

  res
    .status(500)
    .json({
      message:
        "Internal server error",
    });
}
```

This is the simplest version.

---

## 5. Custom application errors

Example:

```js
class AppError
  extends Error {
  constructor(
    message,
    statusCode,
    code
  ) {
    super(message);

    this.statusCode =
      statusCode;

    this.code = code;

    this.isOperational =
      true;
  }
}
```

Usage:

```js
throw new AppError(
  "User not found",
  404,
  "USER_NOT_FOUND"
);
```

---

## 6. Error mapping

Central handler:

```js
function errorHandler(
  err,
  req,
  res,
  next
) {
  const statusCode =
    err.statusCode ?? 500;

  const code =
    err.code ??
    "INTERNAL_ERROR";

  res
    .status(statusCode)
    .json({
      error: {
        code,
        message:
          statusCode >= 500
            ? "Internal server error"
            : err.message,
      },
    });
}
```

This prevents internal details from leaking.

---

## 7. Operational vs programmer errors

Operational errors:

- validation failure
- user not found
- duplicate email
- unauthorized
- external service timeout

Programmer errors:

- undefined property access
- broken assumption
- impossible state
- coding bug

They should not always be treated identically.

---

## 8. Validation errors

Example:

```text
422 Unprocessable Content
```

Response:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request",
    "details": []
  }
}
```

Do not expose library-specific internal objects unnecessarily.

---

## 9. Database errors

Database:

```text
duplicate key
connection failure
timeout
```

Application error mapping:

```text
duplicate email
  -> 409

DB unavailable
  -> 503 or internal handling depending architecture
```

Avoid returning raw SQL/database messages.

---

## 10. JSON parse errors

Malformed JSON should generally become a client error.

Example:

```text
400 Bad Request
```

not:

```text
500 Internal Server Error
```

Your error middleware can detect parser errors and map them appropriately.

---

## 11. Logging

A production error log might include:

```text
requestId
method
route
userId
error code
stack trace
timestamp
```

Do not log:
- passwords
- tokens
- secrets
- payment credentials

---

## 12. Request ID integration

Example:

```js
logger.error({
  requestId:
    req.requestId,
  err,
});
```

Then the client response may include:

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "requestId": "..."
  }
}
```

This makes support/debugging much easier.

---

## 13. headersSent

Sometimes an error occurs after response streaming has begun.

Check:

```js
if (
  res.headersSent
) {
  return next(err);
}
```

This lets Express/default handling deal with the already-started response instead of trying to send new headers.

---

## 14. Do not leak stack traces

Bad:

```js
res
  .status(500)
  .json({
    error:
      err.stack,
  });
```

Potential exposure:
- source paths
- dependencies
- DB details
- implementation internals

Keep stacks in server logs.

---

## 15. 404 is not automatically an error

If no route matches, create a final normal middleware:

```js
app.use(
  (req, res, next) => {
    next(
      new AppError(
        "Route not found",
        404,
        "ROUTE_NOT_FOUND"
      )
    );
  }
);
```

Then centralized error middleware formats the result.

---

## 16. Async errors and Express version awareness

Modern Express versions have improved Promise/async handler behavior compared with older versions.

However, interview-safe knowledge is:

> Every async failure must reliably reach the centralized error handler.

Depending on project version and conventions, that may happen through automatic rejected-Promise forwarding or explicit `next(error)` / wrappers.

Do not assume behavior without knowing the Express version.

---

## 17. Error-handling architecture

```text
Repository
   |
   | DB errors
   v
Service
   |
   | domain translation
   v
Controller
   |
   | next(error)
   v
Error Middleware
   |
   +--> log
   +--> status mapping
   +--> safe response
```

---

## 18. Error response consistency

Example standard shape:

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found",
    "requestId": "abc-123"
  }
}
```

Benefits:
- frontend consistency
- easier logs
- easier testing
- easier API documentation

---

## 19. Common mistakes

### Mistake 1
Different error formats per controller.

### Mistake 2
Returning raw DB errors.

### Mistake 3
Returning stack traces.

### Mistake 4
Registering error middleware before routes.

### Mistake 5
Forgetting the four-argument signature.

### Mistake 6
Trying to send another response after headers were already sent.

---

## 20. Interview questions

### What makes middleware an error handler in Express?

Its four-argument signature: `(err, req, res, next)`.

### Where should error middleware be registered?

After routes and normal middleware.

### Why centralize errors?

For consistent responses, logging, security, and reduced duplication.

### Should database errors be returned directly?

No.

### What if headers are already sent?

Delegate with `next(err)` rather than trying to write new headers.

---

## 21. Strong interview answer

> Express error-handling middleware uses the signature (err, req, res, next) and should be registered after routes. I propagate errors to it, map operational or domain errors to appropriate status codes, log internal details with request context, and return a consistent sanitized response. I never expose raw stack traces or database errors to clients, and I check res.headersSent before attempting to write an error response.

---

## Interview-Ready Summary

```text
Error occurs
   |
   v
next(error)
   |
   v
Error Middleware
   |
   +--> classify
   +--> log
   +--> map status
   +--> sanitize
   +--> respond

Signature:
(err, req, res, next)

Register last.
```

## Section 8 Progress Map

```text
Express
   |
   +--> setup
   +--> routing
   +--> params/query/body
   +--> middleware architecture
   +--> built-in middleware
   +--> custom middleware
   +--> router middleware
   +--> error middleware
   |
   +--> next:
          async error handling
          Express Router
          API versioning
          large application structure
```

## Practice Task

Build a production-style error system with:

- AppError class
- 404 middleware
- validation error mapping
- duplicate-resource error mapping
- request IDs
- centralized structured logging
- sanitized 500 responses
