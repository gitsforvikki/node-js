# Lesson 37 — Error Handling with Async Code

## Why this lesson is critical

Asynchronous error handling is one of the biggest differences between beginner Node.js code and production-ready Node.js code.

You need to understand errors in:

- callbacks
- promises
- async/await
- API handlers
- background tasks
- fire-and-forget operations
- global process-level failures

---

## 1. Synchronous error handling

Synchronous code can be handled with try/catch.

```js
try {
  JSON.parse("{invalid}");
} catch (error) {
  console.error(error.message);
}
```

The error occurs while execution is still inside the try block.

---

## 2. Why try/catch does not catch later callbacks

```js
try {
  setTimeout(() => {
    throw new Error("Boom");
  }, 100);
} catch (error) {
  console.log("Not caught");
}
```

The outer try/catch finishes before the timer callback executes.

Timeline:

```text
try starts
   |
schedule callback
   |
try ends
   |
100 ms later
   |
callback runs
   |
error thrown
```

The original try/catch is no longer active.

---

## 3. Callback error handling

Classic Node-style:

```js
fs.readFile("data.txt", "utf8", (err, data) => {
  if (err) {
    console.error(err);
    return;
  }

  console.log(data);
});
```

Errors are delivered through the callback.

---

## 4. Promise error handling

```js
getUser()
  .then((user) => {
    return getOrders(user.id);
  })
  .catch((error) => {
    console.error(error);
  });
```

A rejection propagates down the chain until handled.

---

## 5. Throw inside promise chain

```js
Promise.resolve("data")
  .then(() => {
    throw new Error("Validation failed");
  })
  .catch((error) => {
    console.log(error.message);
  });
```

Thrown errors become rejected promises.

---

## 6. async/await error handling

```js
async function run() {
  try {
    const user = await getUser();

    return user;
  } catch (error) {
    console.error(error);

    throw error;
  }
}
```

A rejected Promise at `await` behaves like a thrown exception.

---

## 7. Do not swallow errors

Bad:

```js
try {
  await processPayment();
} catch (error) {
  console.log(error);
}
```

If the caller must know the operation failed, simply logging is not enough.

Better:

```js
try {
  await processPayment();
} catch (error) {
  logger.error(error);

  throw error;
}
```

Or translate it into a domain-specific error.

---

## 8. Error translation

Low-level error:

```text
ECONNREFUSED
```

You may want the application layer to throw:

```js
throw new ServiceUnavailableError(
  "Payment provider unavailable"
);
```

This avoids leaking infrastructure details to clients.

---

## 9. Express async error handling

A typical route:

```js
async function getUser(req, res, next) {
  try {
    const user = await userService.getById(
      req.params.id
    );

    res.json(user);
  } catch (error) {
    next(error);
  }
}
```

Then centralized middleware:

```js
function errorHandler(err, req, res, next) {
  console.error(err);

  res.status(500).json({
    message: "Internal server error",
  });
}
```

Framework versions and libraries can provide additional async error behavior, but understanding explicit propagation is essential.

---

## 10. Custom error classes

```js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);

    this.statusCode = statusCode;
    this.isOperational = true;
  }
}
```

Example:

```js
throw new AppError(
  "User not found",
  404
);
```

This supports centralized error translation.

---

## 11. Operational vs programmer errors

### Operational errors

Expected runtime failures.

Examples:
- user not found
- validation failure
- database unavailable
- timeout
- external API failure

These should usually be handled gracefully.

### Programmer errors

Bugs.

Examples:
- reading property of undefined
- incorrect assumptions
- broken invariants
- invalid code path

These may indicate the process is in an unsafe state.

This distinction matters greatly in production error strategy.

---

## 12. Unhandled Promise rejection

Example:

```js
async function run() {
  throw new Error("Failure");
}

run();
```

If the returned promise is not handled, you have an unhandled rejection.

Always ensure async entry points are observed.

```js
run().catch((error) => {
  console.error(error);
});
```

---

## 13. Promise.all error behavior

```js
await Promise.all([
  taskA(),
  taskB(),
  taskC(),
]);
```

If one rejects, `Promise.all` rejects.

Important:

Other already-started operations do not automatically get cancelled.

This is a very important interview point.

```text
task A running
task B rejects
task C running

Promise.all rejects

A and C may still continue
```

Cancellation requires explicit support such as an AbortSignal or library-specific mechanism.

---

## 14. Promise.allSettled for partial failure

If every result matters:

```js
const results = await Promise.allSettled([
  sendEmail(),
  sendSms(),
  pushNotification(),
]);
```

Each result tells you whether it fulfilled or rejected.

Useful when one failure should not discard other outcomes.

---

## 15. Fire-and-forget danger

Bad:

```js
sendAnalyticsEvent();
```

If it returns a Promise and rejects, nobody handles it.

Safer:

```js
void sendAnalyticsEvent().catch((error) => {
  logger.error(
    { error },
    "Analytics event failed"
  );
});
```

Be deliberate when launching work without awaiting it.

---

## 16. finally and cleanup

```js
const connection = await getConnection();

try {
  await processData(connection);
} finally {
  connection.release();
}
```

`finally` is ideal for cleanup that must occur whether the operation succeeds or fails.

---

## 17. Do not leak internal errors to clients

Bad:

```js
res.status(500).json({
  error: error.stack,
});
```

This may expose:
- file paths
- library versions
- SQL details
- secrets
- internal architecture

Production APIs should log internal details securely and return safe messages.

---

## 18. Logging errors properly

Bad:

```js
console.log("error");
```

Better structured logging:

```js
logger.error({
  err: error,
  requestId,
  userId,
});
```

Useful context:
- request ID
- user ID
- route
- external service
- operation
- stack trace

Never log passwords or secrets.

---

## 19. Global process handlers

Node.js exposes process-level events such as:

```js
process.on("unhandledRejection", (reason) => {
  console.error(reason);
});

process.on("uncaughtException", (error) => {
  console.error(error);
});
```

These are not substitutes for local error handling.

They are last-resort observability/shutdown boundaries.

For severe uncaught failures, production systems often log the error, stop accepting traffic, clean up where possible, and restart through a supervisor/container orchestrator.

---

## 20. Error handling layers

A strong production mental model:

```text
Repository
   |
   | low-level DB errors
   v
Service
   |
   | translate business failures
   v
Controller
   |
   | pass error upward
   v
Global Error Middleware
   |
   +--> log
   +--> map status
   +--> sanitize response
   +--> attach request ID
```

Each layer has a responsibility.

---

## 21. Common mistakes

### Mistake 1 — Empty catch blocks

```js
catch (error) {}
```

This hides failures.

### Mistake 2 — Logging and continuing when state is invalid

May corrupt workflows.

### Mistake 3 — Catching everything too early

You may lose context or prevent centralized handling.

### Mistake 4 — Returning stack traces to clients

Security problem.

### Mistake 5 — Forgetting to handle fire-and-forget promises

Creates unhandled rejections.

### Mistake 6 — Assuming Promise.all cancels remaining work

It does not automatically.

---

## 22. Interview questions

### How do you handle errors with async/await?

Use try/catch around awaited operations, then either handle the error, translate it, or rethrow it to a higher-level boundary.

### Can try/catch catch errors from asynchronous callbacks?

Only if the error occurs within an awaited Promise or within the same synchronous call stack. An outer try/catch cannot catch an error thrown later in an unrelated callback.

### What happens when one Promise in Promise.all rejects?

Promise.all rejects with that reason, but other already-started operations may continue.

### What is an unhandled rejection?

A Promise rejection that has no rejection handler attached.

### Should uncaughtException be used to continue the process normally?

Generally no. It indicates an uncaught failure and should be treated as a serious process-level problem.

---

## 23. Strong interview answer

> In Node.js, error handling depends on the async abstraction. Error-first callbacks pass errors explicitly, Promise chains propagate rejections to catch handlers, and async/await allows rejected Promises to be handled with try/catch. In production applications, errors should be handled at appropriate boundaries: low-level infrastructure errors can be translated into domain errors, controllers should forward them to centralized middleware, and internal details should be logged rather than exposed to clients. Unhandled rejections and uncaught exceptions are last-resort process-level failures, not normal control flow.

---

## Interview-Ready Summary

```text
Callbacks
  -> error-first handling

Promises
  -> rejection
  -> catch()

async/await
  -> try/catch

Production strategy
  -> translate
  -> propagate
  -> centralize
  -> log safely
  -> sanitize client response

Promise.all
  -> rejects on first rejection
  -> does NOT automatically cancel others

Global handlers
  -> last-resort process boundary
```

## Section 4 Mental Model

```text
Asynchronous Node.js
        |
        +--> sync vs async
        |
        +--> callbacks
        |      |
        |      +--> error-first convention
        |
        +--> callback hell
        |
        +--> promises
        |      |
        |      +--> pending
        |      +--> fulfilled
        |      +--> rejected
        |
        +--> async/await
        |
        +--> error propagation
               |
               +--> catch
               +--> try/catch
               +--> centralized handling
```

This is the foundation for the next major topic: the Node.js event loop and its execution phases.

## Practice Task

Build an async API flow:

```text
load user
   |
   +--> load profile
   +--> load notifications
   +--> load orders
```

Requirements:
- independent tasks execute concurrently
- failures are handled centrally
- one intentionally failing task is tested
- no unhandled promise rejection occurs
