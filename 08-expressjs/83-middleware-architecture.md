# Lesson 83 — Middleware Architecture

## Why middleware is one of the most important Express concepts

Middleware is the heart of Express.

A strong Express developer should be able to explain:

- what middleware is
- how the request pipeline works
- what `next()` does
- how middleware order affects behavior
- how middleware can short-circuit a request
- how errors move through middleware
- where authentication, validation, logging, and rate limiting belong

---

## 1. What is middleware?

Middleware is a function that runs during the request-response lifecycle.

Typical signature:

```js
function middleware(
  req,
  res,
  next
) {
  // do something

  next();
}
```

It receives:

- `req` — request
- `res` — response
- `next` — function used to continue processing

---

## 2. Request pipeline mental model

```text
Incoming Request
      |
      v
  Logger
      |
      v
Authentication
      |
      v
 Validation
      |
      v
 Controller
      |
      v
  Response
```

Each step can:

- inspect the request
- modify the request
- modify the response
- stop the request
- continue to the next step
- forward an error

---

## 3. The middleware chain

Example:

```js
app.use(logger);

app.use(authenticate);

app.get(
  "/profile",
  validateProfileRequest,
  getProfile
);
```

Execution:

```text
logger
  |
  v
authenticate
  |
  v
validateProfileRequest
  |
  v
getProfile
```

---

## 4. next()

`next()` passes control to the next matching middleware or route handler.

Example:

```js
function logger(
  req,
  res,
  next
) {
  console.log(
    req.method,
    req.url
  );

  next();
}
```

If `next()` is never called and no response is sent, the request hangs.

---

## 5. Short-circuiting middleware

Middleware does not always call `next()`.

Example:

```js
function authenticate(
  req,
  res,
  next
) {
  if (
    !req.headers.authorization
  ) {
    return res
      .status(401)
      .json({
        message:
          "Unauthorized",
      });
  }

  next();
}
```

Flow:

```text
Request
  |
  v
authenticate
  |
  +--> invalid -> response ends
  |
  +--> valid -> next()
```

---

## 6. Middleware order matters

Example:

```js
app.use(
  express.json()
);

app.use(
  requestLogger
);

app.use(
  "/api",
  apiRouter
);
```

This order means:

1. parse JSON
2. log request
3. route request

If body-dependent middleware runs before `express.json()`, `req.body` may not be available.

---

## 7. Global middleware

Registered with:

```js
app.use(middleware);
```

Example:

```js
app.use(requestLogger);
```

It can affect every matching request.

Typical global middleware:

- logging
- body parsers
- CORS
- security headers
- request IDs

---

## 8. Route-level middleware

Route-level middleware is attached to a **specific endpoint**. It runs only when that route matches.

Suppose anyone may list or create users, but deleting a user requires authentication.

### Example: Protect only DELETE

```js
// middlewares/auth.middleware.js
export function authenticate(req, res, next) {
  const authorization = req.headers.authorization;

  if (!authorization) {
    return res.status(401).json({
      message: "Authorization required",
    });
  }

  // Demo only: a header existing does NOT mean the user is authenticated.
  // In production, verify the token and check permissions.
  next();
}
```

```js
// routes/user.routes.js
import { Router } from "express";
import { authenticate } from "../middlewares/auth.middleware.js";
import { getUsers, createUser, deleteUser } from "../controllers/user.controller.js";

const router = Router();

router.get("/", getUsers);                          // Public
router.post("/", createUser);                       // Public
router.delete("/:id", authenticate, deleteUser);     // Protected

export default router;
```

Assuming `app.use("/api/users", router)`, a `DELETE /api/users/1` request flows through:

```text
DELETE /api/users/1
       |
       v
authenticate(req, res, next)
       |
       +-- missing header --> 401 response (stops here)
       |
       +-- header present --> next()
                              |
                              v
                           deleteUser()
```

The `GET` and `POST` routes above **do not run** `authenticate`. A route can also have several middleware functions, called in order:

```js
router.post("/", authenticate, validateUser, createUser);
```

Here `authenticate` runs first, `validateUser` next, and then the controller—provided each middleware calls `next()`.

**Use route-level middleware when one operation needs special validation, authorization, or rate limits.**

---

## 9. Router-level middleware

Router-level middleware is registered on an Express `Router` with `router.use()`. It can run for **multiple routes defined after it**, rather than being repeated on each endpoint.

### Example: Authenticate all user endpoints

Reuse the `authenticate` middleware defined in Section 8.

```js
// routes/user.routes.js
import { Router } from "express";
import { authenticate } from "../middlewares/auth.middleware.js";
import {
  getUsers,
  createUser,
  updateUser,
  deleteUser,
} from "../controllers/user.controller.js";

const router = Router();

// Shared middleware: applies to matching requests that reach this router.
router.use(authenticate);

router.get("/", getUsers);
router.post("/", createUser);
router.put("/:id", updateUser);
router.delete("/:id", deleteUser);

export default router;
```

```js
// app.js
app.use("/api/users", router);
```

Now all four endpoints go through the **same** authentication middleware:

```text
GET    /api/users ----+
POST   /api/users ----+
PUT    /api/users/:id +--> authenticate() --> controller
DELETE /api/users/:id +
```

If the authentication middleware returns `401`, no controller runs. If it calls `next()`, Express continues matching routes in order.

### Combine router-level and route-level middleware

A common production pattern is **authenticate all routes**, but require **admin permission only for deletion**:

```js
router.use(authenticate);                    // Router-level
router.get("/", getUsers);
router.post("/", createUser);
router.delete("/:id", requireAdmin, deleteUser); // Route-level
```

```text
DELETE /api/users/1
       |
       v
authenticate()  [router-level]
       |
       v
requireAdmin()  [route-level]
       |
       v
deleteUser()    [controller]
```

`requireAdmin` must check the role of the **verified** user, not trust a role supplied directly by the client.

### Key differences

| Point | Route-level | Router-level |
|---|---|---|
| Registration | `router.delete("/:id", middleware, handler)` | `router.use(middleware)` |
| Scope | One route definition | Matching requests reaching the router after registration |
| Use case | DELETE-only admin check | Shared authentication or logging |
| Multiple middleware | Yes | Yes |

**Important:** Registration order matters. If a route handles the request before `router.use(authenticate)`, that earlier route is not retroactively protected. Router-level middleware may also be restricted to a path, such as `router.use("/admin", authenticate)`.

**Interview takeaway:** Route-level middleware targets an individual endpoint, while `router.use()` shares middleware across a router's matching requests. Both participate in Express's ordered middleware pipeline.

This is covered in more depth in Lesson 86.

---

## 10. Middleware can enrich req

Example:

```js
function authenticate(
  req,
  res,
  next
) {
  req.user = {
    id: 123,
    role: "admin",
  };

  next();
}
```

Later:

```js
function getProfile(
  req,
  res
) {
  res.json(
    req.user
  );
}
```

This is common for:
- user identity
- request ID
- validated payload
- permissions

---

## 11. Prefer explicit names for enriched data

Instead of scattering raw changes everywhere:

```js
req.foo = ...
req.bar = ...
```

use clear conventions such as:

```js
req.user
req.validatedBody
req.requestId
```

This improves maintainability.

---

## 12. Synchronous middleware

```js
function middleware(
  req,
  res,
  next
) {
  req.startTime =
    Date.now();

  next();
}
```

---

## 13. Asynchronous middleware

```js
async function authenticate(
  req,
  res,
  next
) {
  try {
    const user =
      await verifyToken(
        req.headers.authorization
      );

    req.user = user;

    next();
  } catch (error) {
    next(error);
  }
}
```

Async middleware errors must be propagated correctly.

---

## 14. next(error)

Calling:

```js
next(error);
```

skips normal middleware and routes toward error-handling middleware.

Conceptually:

```text
Normal middleware
      |
      v
error occurs
      |
      v
next(error)
      |
      v
Error middleware
```

---

## 15. Middleware responsibilities

Good middleware responsibilities are narrow.

Examples:

```text
authenticate
authorize
validate
rateLimit
requestLogger
cors
requestId
```

Bad middleware:

```text
authenticate
+ query DB
+ process payment
+ send email
+ update inventory
```

That mixes concerns.

---

## 16. Middleware should not contain core business logic

Business flow belongs in services.

Middleware is best for request lifecycle concerns.

Example:

```text
Middleware
  -> authentication

Controller
  -> HTTP translation

Service
  -> business logic

Repository
  -> data access
```

---

## 17. Middleware stack architecture

A production-style Express stack might look like:

```text
Request
   |
   v
Request ID
   |
   v
Logging
   |
   v
Security Headers
   |
   v
CORS
   |
   v
Body Parser
   |
   v
Rate Limiter
   |
   v
Router
   |
   +--> Auth
   +--> Validation
   +--> Controller
   |
   v
404
   |
   v
Error Handler
```

---

## 18. Response lifecycle middleware

Sometimes middleware needs to observe the completed response.

Example:

```js
function timing(
  req,
  res,
  next
) {
  const start =
    Date.now();

  res.on(
    "finish",
    () => {
      console.log(
        Date.now() - start
      );
    }
  );

  next();
}
```

Useful for:
- request timing
- logging
- metrics

---

## 19. Common mistakes

### Mistake 1
Calling `next()` after sending a final response.

This can create double-response bugs.

### Mistake 2
Forgetting `next()`.

The request hangs.

### Mistake 3
Wrong middleware order.

### Mistake 4
Putting heavy business logic in middleware.

### Mistake 5
Ignoring async errors.

---

## 20. Interview questions

### What is Express middleware?

A function in the request-response pipeline that can inspect or modify req/res, terminate the response, or pass control onward using next().

### What happens if next() is not called?

Processing stops unless that middleware sends a response.

### What does next(error) do?

It routes control to error-handling middleware.

### Why does middleware order matter?

Express executes middleware in registration order for matching requests.

---

## 21. Strong interview answer

> Middleware is the core pipeline abstraction in Express. Each middleware receives req, res, and next, and can inspect or modify the request, terminate the response, or continue to the next middleware. Order is significant because middleware executes sequentially. I use middleware for cross-cutting request concerns such as authentication, validation, logging, rate limiting, and request IDs, while keeping business logic in the service layer.

---

## Interview-Ready Summary

```text
Middleware
   |
   +--> req
   +--> res
   +--> next()

Can:
   inspect
   modify
   stop
   continue
   forward errors

Order matters.

Use middleware for:
auth
validation
logging
security
rate limiting
```

## Practice Task

Build a route:

```text
POST /orders
```

with middleware sequence:

1. request ID
2. logger
3. authenticate
4. validate body
5. controller
6. centralized error handler
