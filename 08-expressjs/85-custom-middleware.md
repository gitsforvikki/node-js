# Lesson 85 — Custom Middleware

## Why custom middleware matters

Built-in middleware solves generic HTTP concerns.

Custom middleware lets you implement application-specific request pipeline behavior such as:

- authentication
- authorization
- validation
- request IDs
- logging
- tenant resolution
- feature flags
- API-key checks

---

## 1. Basic custom middleware

```js
function logger(
  req,
  res,
  next
) {
  console.log(
    req.method,
    req.originalUrl
  );

  next();
}
```

Register:

```js
app.use(logger);
```

---

## 2. Request ID middleware

```js
import crypto
  from "node:crypto";

function requestId(
  req,
  res,
  next
) {
  req.requestId =
    crypto.randomUUID();

  res.setHeader(
    "X-Request-Id",
    req.requestId
  );

  next();
}
```

This helps correlate logs across a request.

---

## 3. Authentication middleware

```js
async function authenticate(
  req,
  res,
  next
) {
  try {
    const token =
      extractToken(req);

    if (!token) {
      return res
        .status(401)
        .json({
          message:
            "Unauthorized",
        });
    }

    const user =
      await verifyToken(
        token
      );

    req.user = user;

    next();
  } catch (error) {
    next(error);
  }
}
```

---

## 4. Authorization middleware

Authentication asks:

```text
Who are you?
```

Authorization asks:

```text
Are you allowed?
```

Example:

```js
function requireAdmin(
  req,
  res,
  next
) {
  if (
    req.user?.role !==
    "admin"
  ) {
    return res
      .status(403)
      .json({
        message:
          "Forbidden",
      });
  }

  next();
}
```

---

## 5. Middleware factory

A middleware function can be generated dynamically.

Example:

```js
function requireRole(
  ...allowedRoles
) {
  return (
    req,
    res,
    next
  ) => {
    if (
      !allowedRoles.includes(
        req.user?.role
      )
    ) {
      return res
        .status(403)
        .json({
          message:
            "Forbidden",
        });
    }

    next();
  };
}
```

Usage:

```js
router.delete(
  "/:id",
  authenticate,
  requireRole(
    "admin"
  ),
  deleteUser
);
```

---

## 6. Validation middleware

Conceptually:

```js
function validateBody(
  schema
) {
  return (
    req,
    res,
    next
  ) => {
    const result =
      schema.safeParse(
        req.body
      );

    if (!result.success) {
      return res
        .status(422)
        .json({
          errors:
            result.error.issues,
        });
    }

    req.validatedBody =
      result.data;

    next();
  };
}
```

This removes raw parsing concerns from controllers.

---

## 7. Timing middleware

```js
function timing(
  req,
  res,
  next
) {
  const start =
    process.hrtime.bigint();

  res.on(
    "finish",
    () => {
      const end =
        process.hrtime.bigint();

      const ms =
        Number(
          end - start
        ) / 1e6;

      console.log(
        req.method,
        req.originalUrl,
        ms
      );
    }
  );

  next();
}
```

Useful for observability.

---

## 8. Middleware should be reusable

Bad:

```js
function createUserMiddleware(
  req,
  res,
  next
) {
  // create user
  // send email
  // update analytics
  // validate
  // authorize
}
```

Too many responsibilities.

Good custom middleware is focused.

---

## 9. Do not hide too much business logic

Middleware should support the request lifecycle.

Business operations belong in services.

Bad:

```text
middleware:
charge payment
reserve stock
create order
```

Better:

```text
middleware:
authenticate
validate

service:
checkout workflow
```

---

## 10. Async middleware errors

If async code throws, make sure the error reaches centralized handling.

Pattern:

```js
async function middleware(
  req,
  res,
  next
) {
  try {
    await something();

    next();
  } catch (error) {
    next(error);
  }
}
```

Depending on Express version and project patterns, async error behavior may differ, so keep your error strategy consistent.

---

## 11. Avoid next after response

Bad:

```js
if (!req.user) {
  res
    .status(401)
    .json({
      message:
        "Unauthorized",
    });

  next();
}
```

This can continue into later middleware.

Correct:

```js
if (!req.user) {
  return res
    .status(401)
    .json({
      message:
        "Unauthorized",
    });
}
```

---

## 12. Mutating req

Custom middleware often attaches data.

Examples:

```js
req.user
req.requestId
req.validatedBody
req.permissions
```

Be consistent and document these conventions.

In TypeScript, extend request types properly.

---

## 13. Middleware ordering example

```js
router.post(
  "/orders",
  authenticate,
  loadPermissions,
  validateOrder,
  createOrder
);
```

Order matters:

```text
authenticate
   |
   v
permissions
   |
   v
validation
   |
   v
controller
```

---

## 14. Testing middleware

Test:
- success path
- rejected auth
- invalid input
- missing headers
- next() called correctly
- no double responses
- async error propagation

---

## 15. Common mistakes

### Mistake 1
Middleware doing too much.

### Mistake 2
Calling next after response.

### Mistake 3
Forgetting next.

### Mistake 4
Ignoring async errors.

### Mistake 5
Attaching arbitrary undocumented fields to req.

---

## 16. Interview questions

### What is custom middleware?

Application-defined middleware that runs in the Express request pipeline.

### What is a middleware factory?

A function that returns configured middleware.

### Why use validation middleware?

To keep controllers clean and ensure only validated input reaches business logic.

### Authentication vs authorization middleware?

Authentication identifies the caller; authorization checks permissions.

---

## 17. Strong interview answer

> Custom middleware is how I implement reusable cross-cutting request concerns such as authentication, authorization, validation, request IDs, and logging. I keep middleware focused, avoid putting core business logic inside it, use middleware factories for configurable behavior such as role checks, and make sure async errors are propagated to centralized error handling.

---

## Interview-Ready Summary

```text
Custom Middleware
   |
   +--> auth
   +--> authorization
   +--> validation
   +--> request ID
   +--> logging
   +--> timing

Good middleware:
focused
reusable
predictable
error-aware
```

## Practice Task

Build these middleware functions:

- requestId
- authenticate
- requireRole("admin")
- validateBody(schema)
- timing

Apply them to a protected admin route.
