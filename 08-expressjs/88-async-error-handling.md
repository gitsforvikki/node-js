# Lesson 88 — Async Error Handling

## Why this lesson matters

Async error handling is one of the most important Express interview topics.

Most real Express applications use:

- async/await
- database queries
- external APIs
- authentication services
- queues
- file operations

If async failures do not reach centralized error middleware, your API can become inconsistent, leak rejections, or crash.

---

## 1. Synchronous errors

Express can catch many synchronous throws inside route handlers.

Example:

```js
app.get(
  "/sync",
  (req, res) => {
    throw new Error(
      "Something failed"
    );
  }
);
```

This error can flow into error middleware.

---

## 2. Async errors are different

Consider:

```js
app.get(
  "/users",
  async (req, res) => {
    const users =
      await userService.list();

    res.json(users);
  }
);
```

If `userService.list()` rejects, that rejection must reach Express's error-handling pipeline.

---

## 3. Express 4 pattern

Historically, with Express 4, a common safe pattern was:

```js
app.get(
  "/users",
  async (
    req,
    res,
    next
  ) => {
    try {
      const users =
        await userService.list();

      res.json(users);
    } catch (error) {
      next(error);
    }
  }
);
```

This guarantees the async error reaches error middleware.

---

## 4. Async wrapper pattern

To avoid repeating try/catch everywhere:

```js
function asyncHandler(
  handler
) {
  return function (
    req,
    res,
    next
  ) {
    Promise
      .resolve(
        handler(
          req,
          res,
          next
        )
      )
      .catch(next);
  };
}
```

Usage:

```js
router.get(
  "/users",
  asyncHandler(
    async (
      req,
      res
    ) => {
      const users =
        await userService.list();

      res.json(users);
    }
  )
);
```

---

## 5. Why wrappers became popular

Without wrapper:

```text
controller A
  -> try/catch

controller B
  -> try/catch

controller C
  -> try/catch
```

With wrapper:

```text
async handler
   |
   v
Promise rejection
   |
   v
next(error)
   |
   v
central error middleware
```

This reduces repetition.

---

## 6. Express 5 behavior

Modern Express 5 improves rejected Promise handling.

If an async route handler returns a Promise that rejects, Express can forward that rejection to error middleware.

Example:

```js
app.get(
  "/users",
  async (req, res) => {
    const users =
      await userService.list();

    res.json(users);
  }
);
```

If `list()` rejects, Express 5 can route the error automatically.

---

## 7. Interview-safe explanation

A strong answer is:

> In older Express 4 codebases, rejected async handlers usually need explicit `next(error)` or a wrapper. In Express 5, rejected Promises returned by handlers are forwarded to the error pipeline automatically.

This shows version awareness.

---

## 8. Fire-and-forget async work

Bad:

```js
app.post(
  "/orders",
  async (req, res) => {
    sendAnalytics();

    res.status(201).json({
      success: true,
    });
  }
);
```

If `sendAnalytics()` returns a Promise and rejects, the error may be unhandled.

Safer:

```js
void sendAnalytics()
  .catch((error) => {
    logger.error(error);
  });
```

Or move non-critical background work to a queue.

---

## 9. Do not swallow errors

Bad:

```js
try {
  await createOrder();
} catch (error) {
  console.log(error);
}
```

The client may never know the request failed.

Better:

```js
try {
  await createOrder();
} catch (error) {
  next(error);
}
```

or throw a mapped application error.

---

## 10. Error translation

Repository:

```text
duplicate key error
```

Service:

```text
EMAIL_ALREADY_EXISTS
```

Controller/error layer:

```text
409 Conflict
```

This is cleaner than leaking DB-specific details.

---

## 11. Async middleware

Same rule applies to middleware.

```js
async function authenticate(
  req,
  res,
  next
) {
  const user =
    await verifySession();

  req.user = user;

  next();
}
```

If `verifySession()` rejects, make sure your Express version/pattern reliably forwards it.

---

## 12. Async validation

Example:

```js
async function validateUniqueEmail(
  req,
  res,
  next
) {
  try {
    const exists =
      await userRepository.existsByEmail(
        req.body.email
      );

    if (exists) {
      return res
        .status(409)
        .json({
          message:
            "Email exists",
        });
    }

    next();
  } catch (error) {
    next(error);
  }
}
```

---

## 13. Promise chain alternative

```js
app.get(
  "/users",
  (req, res, next) => {
    userService
      .list()
      .then((users) => {
        res.json(users);
      })
      .catch(next);
  }
);
```

Works, but async/await is usually easier to read.

---

## 14. Unhandled rejection danger

If async failures escape the request pipeline, you may see:

```text
UnhandledPromiseRejection
```

or process-level rejection behavior.

The request may also hang or return an incorrect response.

---

## 15. Double response after async failure

Bad:

```js
try {
  await doWork();

  res.json({
    success: true,
  });
} catch (error) {
  next(error);
}

res.end();
```

This can attempt another response.

Always structure returns/control flow carefully.

---

## 16. Common mistakes

### Mistake 1
Assuming all Express versions automatically handle rejected async handlers.

### Mistake 2
Catching errors and only logging them.

### Mistake 3
Ignoring fire-and-forget rejections.

### Mistake 4
Sending a response and then continuing execution.

### Mistake 5
Returning raw infrastructure errors directly.

---

## 17. Interview questions

### How do you handle async errors in Express?

Ensure rejected Promises reach centralized error middleware using explicit `next(error)`, an async wrapper, or native Promise forwarding in Express 5.

### Why were async wrappers common?

To avoid repeating try/catch in every async route in Express 4.

### What changed in Express 5?

Promise rejections returned by handlers can be forwarded automatically.

### What is wrong with catching and only logging?

The request flow may continue incorrectly and the client may not receive the proper failure response.

---

## 18. Strong interview answer

> Async error handling in Express is about making sure every rejected Promise reaches the centralized error middleware. In Express 4, teams commonly used try/catch with next(error) or an async wrapper. Express 5 improves this by automatically forwarding rejected Promises returned by route handlers and middleware. I still make sure fire-and-forget work has its own error boundary and that infrastructure errors are translated into safe application errors before responding.

---

## Interview-Ready Summary

```text
Async Failure
    |
    v
next(error)
or Promise rejection forwarding
    |
    v
Error Middleware

Express 4:
wrapper / try-catch common

Express 5:
native rejected-Promise forwarding
```

## Practice Task

Implement the same controller in three ways:

1. manual try/catch
2. asyncHandler wrapper
3. Express 5 native async handler

Explain the differences.
