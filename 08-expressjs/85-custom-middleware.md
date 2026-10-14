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

Validation middleware **checks incoming request data before the controller runs**. It prevents malformed input from reaching business logic and avoids repeating the same validation code in every controller.

For example, a client might send:

```json
{
  "name": "",
  "email": "wrong-email",
  "isActive": "yes"
}
```

The server should reject this instead of creating an invalid user.

### Why use a middleware factory?

`validateBody(schema)` is a **function that returns an Express middleware function**. You pass different schemas to the same reusable validator.

```js
// src/middlewares/validate.middleware.js
export const validateBody = (schema) => {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      return res.status(422).json({
        message: "Validation failed",
        errors: result.error.issues,
      });
    }

    req.validatedBody = result.data;
    next();
  };
};
```

**Line by line:**

1. `validateBody(schema)` receives validation rules.
2. The returned `(req, res, next)` function runs for each matching request.
3. `schema.safeParse(req.body)` checks and parses the incoming body without throwing on ordinary validation failures.
4. If invalid, the middleware sends HTTP `422` with `result.error.issues` and **stops**.
5. If valid, `result.data` is stored in `req.validatedBody`, and `next()` passes control to the controller.

### Complete example with Zod

Install the validator:

```bash
npm install zod
```

Project layout:

```text
express-practice/
├── server.js
└── src/
    ├── middlewares/validate.middleware.js
    ├── validators/user.schema.js
    ├── routes/user.routes.js
    └── controllers/user.controller.js
```

**Step 1 — Define the schema**

```js
// src/validators/user.schema.js
import { z } from "zod";

export const createUserSchema = z.object({
  name: z.string().trim().min(2),
  email: z.email(),
  isActive: z.boolean(),
});
```

This requires a nonempty name with at least 2 characters, a valid email, and a boolean `isActive` (including `false`). `z.email()` is Zod 4 syntax; in Zod 3 use `z.string().email()`.

**Step 2 — Attach middleware to the route**

```js
// src/routes/user.routes.js
import { Router } from "express";
import { validateBody } from "../middlewares/validate.middleware.js";
import { createUserSchema } from "../validators/user.schema.js";
import { createUser, updateUser } from "../controllers/user.controller.js";

const router = Router();

router.post("/", validateBody(createUserSchema), createUser);
router.put("/:id", validateBody(createUserSchema), updateUser);

export default router;
```

**Step 3 — Use validated data in the controller**

```js
// src/controllers/user.controller.js
export const createUser = (req, res) => {
  const user = req.validatedBody;

  // In a real app, call a service to save the user.
  return res.status(201).json({
    message: "User created successfully",
    data: user,
  });
};
```

**Step 4 — Configure Express**

```js
// server.js
import express from "express";
import userRouter from "./src/routes/user.routes.js";

const app = express();

app.use(express.json()); // Must run before validation middleware.
app.use("/api/users", userRouter);

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

### How does the request flow?

```text
POST /api/users
       |
       v
express.json() --> req.body
       |
       v
validateBody(createUserSchema)
       |
       +-- Invalid --> 422 response (controller skipped)
       |
       +-- Valid --> req.validatedBody = result.data
                         |
                         v
                       next()
                         |
                         v
                      createUser()
```

**Why `req.validatedBody`?** `result.data` contains the **parsed** values from Zod. For example, `trim()` can remove spaces from the name, and Zod object schemas normally strip unknown keys. The controller should use `req.validatedBody`, not the original untrusted `req.body`.

### POST vs PUT vs PATCH

- **POST:** Use `validateBody(createUserSchema)` to require all fields.
- **PUT:** You can reuse that schema if the update requires all the same fields. Validate `req.params.id` separately.
- **PATCH:** Often uses `createUserSchema.partial()` to allow individual fields to be updated. Consider rejecting an empty body.

**Interview takeaway:** Validation middleware is a reusable request-pipeline concern. `validateBody(schema)` creates the middleware during route setup; Express invokes the returned function for each request. It either returns a validation error or calls `next()` with parsed data available for the controller.

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
