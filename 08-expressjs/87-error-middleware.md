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


---

## Complete Flow Example — Centralized Error Handling in Express 5

This end-to-end example connects **routes → validation middleware → controller → service → centralized error handler**. It also demonstrates request IDs, logging, 404/409/422 errors, malformed JSON, and safe 500 responses.

### 1. What problem does centralized error handling solve?

Without it, every controller may repeat `try/catch`, `console.error`, and different JSON response formats. With it, the **service throws known application errors**, Express forwards failures, and **one error handler logs and formats responses**.

```text
Request
  |
  v
requestId -> express.json() -> Router
                              |
                              +--> validateBody() -- invalid --> next(error)
                              |
                              +--> Controller -> Service -- throws AppError
                              |                       |
                              +--> success response   |
                                                      v
                                             Error middleware
                                              |    |     |
                                           classify log respond
```

### 2. Install and organize

```bash
npm install express zod
```

Use `"type": "module"` in `package.json` and a script such as `"dev": "node --watch server.js"`.

```text
express-practice/
├── package.json
├── server.js
└── src/
    ├── app.js
    ├── utils/app-error.js
    ├── services/user.service.js
    ├── controllers/user.controller.js
    ├── validators/user.schema.js
    ├── routes/user.routes.js
    └── middlewares/
        ├── request-id.middleware.js
        ├── validate.middleware.js
        ├── not-found.middleware.js
        └── error.middleware.js
```

### 3. Custom application error

`src/utils/app-error.js`

```js
export class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.name = "AppError";
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
    Error.captureStackTrace?.(this, this.constructor);
  }
}
```

Normal `Error` describes a failure; `AppError` additionally carries an HTTP status and a stable code. Expected failures such as missing users are **operational** errors. Unexpected `TypeError` or coding mistakes should not be exposed to clients.

### 4. Service: throw errors, do not send HTTP responses

`src/services/user.service.js`

```js
import { AppError } from "../utils/app-error.js";

const users = [
  { id: 1, name: "Vikash", email: "vikash@gmail.com", isActive: true },
  { id: 2, name: "Rahul", email: "rahul@gmail.com", isActive: false },
];

export const getUsersService = () => users;

export const getUserByIdService = (id) => {
  const user = users.find((user) => user.id === id);
  if (!user) {
    throw new AppError("User not found", 404, "USER_NOT_FOUND");
  }
  return user;
};

export const createUserService = (data) => {
  if (users.some((user) => user.email === data.email)) {
    throw new AppError("Email already exists", 409, "DUPLICATE_EMAIL");
  }

  const user = {
    id: Math.max(0, ...users.map((u) => u.id)) + 1,
    ...data,
  };
  users.push(user);
  return user;
};

export const deleteUserService = (id) => {
  const index = users.findIndex((user) => user.id === id);
  if (index === -1) {
    throw new AppError("User not found", 404, "USER_NOT_FOUND");
  }
  return users.splice(index, 1)[0];
};
```

This is an **in-memory learning example**. A real application would use a database and map known database errors to safe application errors.

### 5. Controllers: normal success responses

`src/controllers/user.controller.js`

```js
import {
  getUsersService,
  getUserByIdService,
  createUserService,
  deleteUserService,
} from "../services/user.service.js";
import { AppError } from "../utils/app-error.js";

const parseId = (value) => {
  const id = Number(value);
  if (!Number.isSafeInteger(id) || id <= 0) {
    throw new AppError("Invalid user ID", 400, "INVALID_USER_ID");
  }
  return id;
};

export const getUsers = async (req, res) => {
  res.status(200).json({ success: true, data: getUsersService() });
};

export const getUserById = async (req, res) => {
  const user = getUserByIdService(parseId(req.params.id));
  res.status(200).json({ success: true, data: user });
};

export const createUser = async (req, res) => {
  const user = createUserService(req.validatedBody);
  res.status(201).json({ success: true, data: user });
};

export const deleteUser = async (req, res) => {
  deleteUserService(parseId(req.params.id));
  res.status(204).send();
};
```

**Why no `try/catch` here?** In **Express 5**, rejected Promises from async route handlers are automatically forwarded to the error handler. For Express 4, use `next(error)` in `catch` or an async wrapper. Detached timers/background tasks are not automatically captured by Express.

### 6. Validation middleware: forward errors centrally

`src/validators/user.schema.js`

```js
import { z } from "zod";

export const createUserSchema = z.object({
  name: z.string().trim().min(2),
  email: z.email(),
  isActive: z.boolean(),
});
```

`z.email()` is Zod 4 syntax; Zod 3 uses `z.string().email()`.

`src/middlewares/validate.middleware.js`

```js
import { AppError } from "../utils/app-error.js";

export const validateBody = (schema) => (req, res, next) => {
  const result = schema.safeParse(req.body);

  if (!result.success) {
    const error = new AppError(
      "Invalid request body",
      422,
      "VALIDATION_ERROR"
    );
    error.details = result.error.issues.map((issue) => ({
      field: issue.path.join("."),
      message: issue.message,
    }));
    return next(error);
  }

  req.validatedBody = result.data;
  next();
};
```

Instead of sending its own error JSON, this middleware uses `next(error)`. That means validation errors get the **same logging and response shape** as service errors. The controller uses `req.validatedBody` because it contains the successfully parsed/transformed input.

### 7. Request ID: correlate client errors with logs

`src/middlewares/request-id.middleware.js`

```js
import { randomUUID } from "node:crypto";

export const requestId = (req, res, next) => {
  req.requestId = randomUUID();
  res.setHeader("X-Request-Id", req.requestId);
  next();
};
```

Each request receives a unique identifier, which appears in both server logs and error responses.

### 8. Unmatched routes: generate a 404

`src/middlewares/not-found.middleware.js`

```js
import { AppError } from "../utils/app-error.js";

export const notFound = (req, res, next) => {
  next(new AppError("Route not found", 404, "ROUTE_NOT_FOUND"));
};
```

An unmatched URL does not automatically throw an error. This final normal middleware forwards an `AppError` into centralized handling.

### 9. Centralized error handler: classify, log, respond

`src/middlewares/error.middleware.js`

```js
import { AppError } from "../utils/app-error.js";

export const errorHandler = (err, req, res, next) => {
  // A response has already started: delegate instead of sending twice.
  if (res.headersSent) return next(err);

  let error = err;

  // JSON body parser failures should be 400, not 500.
  if (err instanceof SyntaxError && err.status === 400 && "body" in err) {
    error = new AppError("Malformed JSON body", 400, "INVALID_JSON");
  }

  const isOperational = error instanceof AppError && error.isOperational;
  const statusCode =
    isOperational && error.statusCode >= 400 && error.statusCode < 600
      ? error.statusCode
      : 500;
  const code = isOperational ? error.code : "INTERNAL_ERROR";

  // Structured server-side log (simple teaching implementation).
  const log = {
    timestamp: new Date().toISOString(),
    level: statusCode >= 500 ? "error" : "warn",
    requestId: req.requestId,
    method: req.method,
    path: req.path,
    statusCode,
    code,
    message: err.message,
    stack: err.stack,
  };

  if (statusCode >= 500) console.error(JSON.stringify(log));
  else console.warn(JSON.stringify(log));

  // Never send stack traces or unexpected internal messages to clients.
  const response = {
    success: false,
    error: {
      code,
      message: isOperational ? error.message : "Internal server error",
      requestId: req.requestId,
    },
  };

  if (isOperational && error.details) {
    response.error.details = error.details;
  }

  return res.status(statusCode).json(response);
};
```

**Why four parameters?** Express recognizes an error middleware by its signature: `(err, req, res, next)`.

**Why `res.headersSent`?** If response headers are already sent, trying to send another JSON response can fail. Delegate to Express's remaining error handling.

**Logging vs response:** Logs may contain error stack traces and debugging context, while clients get a safe, consistent error code/message/request ID. In production use Pino/Winston or your platform's log pipeline, with sensitive-field redaction. Never log passwords, tokens, payment credentials, or raw private request bodies. Expected 4xx errors may be logged at a lower severity or sampled to reduce noise.

### 10. Wire routes and app in the correct order

`src/routes/user.routes.js`

```js
import { Router } from "express";
import {
  getUsers,
  getUserById,
  createUser,
  deleteUser,
} from "../controllers/user.controller.js";
import { validateBody } from "../middlewares/validate.middleware.js";
import { createUserSchema } from "../validators/user.schema.js";

const router = Router();
router.get("/", getUsers);
router.get("/:id", getUserById);
router.post("/", validateBody(createUserSchema), createUser);
router.delete("/:id", deleteUser);

export default router;
```

`src/app.js`

```js
import express from "express";
import userRouter from "./routes/user.routes.js";
import { requestId } from "./middlewares/request-id.middleware.js";
import { notFound } from "./middlewares/not-found.middleware.js";
import { errorHandler } from "./middlewares/error.middleware.js";

const app = express();

app.use(requestId);                // 1. Assign request ID
app.use(express.json());           // 2. Parse JSON (may throw syntax errors)
app.use("/api/users", userRouter); // 3. Normal routes
app.use(notFound);                 // 4. Unknown URLs
app.use(errorHandler);             // 5. Error middleware LAST

export default app;
```

`server.js`

```js
import app from "./src/app.js";

app.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});
```

### 11. Test the complete error flow

Run `npm run dev` and try:

| Request / situation | HTTP | Public error code |
|---|---:|---|
| `GET /api/users/99` | 404 | `USER_NOT_FOUND` |
| `GET /api/users/abc` | 400 | `INVALID_USER_ID` |
| `POST /api/users` with duplicate email | 409 | `DUPLICATE_EMAIL` |
| `POST /api/users` with invalid body | 422 | `VALIDATION_ERROR` |
| `GET /api/unknown` | 404 | `ROUTE_NOT_FOUND` |
| Malformed JSON body | 400 | `INVALID_JSON` |
| Unexpected thrown `TypeError` | 500 | `INTERNAL_ERROR` |

Example invalid body:

```bash
curl -i -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"","email":"wrong-email","isActive":"yes"}'
```

Example expected client response for a missing user (`requestId` will differ):

```json
{
  "success": false,
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found",
    "requestId": "example-request-id"
  }
}
```

Example server log (illustrative, with stack abbreviated):

```json
{
  "timestamp": "2026-10-14T10:30:00.000Z",
  "level": "warn",
  "requestId": "example-request-id",
  "method": "GET",
  "path": "/api/users/99",
  "statusCode": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "stack": "AppError: User not found..."
}
```

The **same request ID** connects the user's response to the matching server log. For a programming error, the server logs the real stack, but the client only receives `INTERNAL_ERROR` and `Internal server error`.

### 12. throw vs next(error) vs res.json()

```js
// Service: describe a known failure.
throw new AppError("User not found", 404, "USER_NOT_FOUND");

// Middleware: explicitly forward a failure.
return next(error);

// Central error handler: send the final HTTP error response.
return res.status(statusCode).json(response);
```

**Interview takeaway:** Services identify failures, Express propagates them, and centralized error middleware classifies, logs, sanitizes, and responds. The result is consistent API errors without repeating error handling in every controller.

