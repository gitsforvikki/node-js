# Lesson 151 — Operational vs Programmer Errors

## 1. Why do we classify errors?

When an API fails, we first ask: **Was this an expected problem in the real world, or a bug in our code?** The answer determines how we respond, log, and recover.

Imagine a restaurant:
- Customer orders an unavailable dish → **operational problem**: respond politely and continue serving.
- Chef accidentally writes a broken recipe → **programmer problem**: fix the recipe; do not pretend it is a normal customer request.

## 2. Operational errors

An **operational error** is a failure your application should expect might occur while running.

Examples:
- Client requests a user that does not exist → 404.
- Client submits invalid data → 400.
- User is not authenticated → 401.
- Upstream payment API times out → 503/504 depending on your role.
- Database connection temporarily fails → service-unavailable behavior.

These errors are not always the client's fault, but they should be **handled deliberately**, without exposing internal details.

```js
app.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }
  res.json(user);
});
```

The missing user is a normal possibility, so it is an operational case.

## 3. Programmer errors

A **programmer error** is a bug or violated code assumption.

```js
app.get("/users", (req, res) => {
  const users = undefined;
  res.json({ count: users.length }); // TypeError: bug
});
```

Examples:
- Accessing `undefined.name`.
- Calling a non-function.
- Incorrect assumptions about object shape.
- An invariant that should never be violated.

**Do not classify every TypeError blindly**: investigate context. A database outage is usually operational, but forgetting to await a Promise is a code bug.

## 4. Why the difference matters

| Question | Operational | Programmer |
| --- | --- | --- |
| Expected during normal operation? | Yes, possible | No, bug |
| API response | Safe message + appropriate status | Generic 500 |
| Log severity | Depends on impact | Usually error/critical |
| Fix | Handle gracefully, retry when safe | Correct the code |
| Can request continue? | Often yes | Current request should fail safely |
| Should process restart? | Usually not necessary | May require restart if state is unreliable |

Don't automatically restart for every 500 or every rejected request. Conversely, do not assume an unexpected process-level crash can be safely ignored.

## 5. Common HTTP codes

| Code | Meaning | Example |
| --- | --- | --- |
| 400 | Bad request | Invalid input |
| 401 | Unauthenticated | Missing/expired credentials |
| 403 | Forbidden | No permission |
| 404 | Not found | Missing product |
| 409 | Conflict | Duplicate email or conflicting state |
| 429 | Too many requests | Rate limit |
| 500 | Internal server error | Unexpected bug |
| 503 | Service unavailable | Required backend temporarily unavailable |

A unique-index error may be mapped to 409 *when your application knows what it means*. Do not return raw MongoDB errors directly.

## 6. Error handling flow in Express 5

```text
Client GET /users/123
   |
   v
Controller -> query DB
   |                  |
   |                  +-- no user -> known 404
   |
   +-- unexpected bug -> rejected async handler
                              |
                              v
                    centralized error handler
                              |
                              v
                   safe JSON response + log
```

Express 5 forwards thrown errors and rejected Promises from **returned async route/middleware handlers** to error middleware.

## 7. Real-world example: checkout

- Invalid quantity → reject with 400.
- Stock is unavailable → respond with 409.
- Payment provider is unavailable → record state, fail/retry safely as appropriate (avoid duplicate charges).
- `cart.total.toFixed()` throws because `total` is undefined → bug: fix application code.

**Important:** An external timeout is operational, but blind retries on payment operations can create duplicate charges. Use provider-supported idempotency and appropriate retry rules.

## 8. How should a production API respond?

Provide helpful public messages for known errors, but don't leak stack traces or secrets:

```json
{
  "success": false,
  "code": "PRODUCT_NOT_FOUND",
  "message": "Product not found"
}
```

For unexpected failures:

```json
{
  "success": false,
  "code": "INTERNAL_ERROR",
  "message": "Internal server error"
}
```

Log the real technical error privately; the API response is not your debugging console.

## 9. Mistakes to avoid

- Catching an exception and responding `200 OK`.
- Sending raw database messages, stack traces, or tokens to clients.
- Calling every error "user error".
- Silently swallowing programmer bugs.
- Retrying every error without considering side effects.
- Restarting the entire service for normal 404s.

## 10. Practice and interview

Classify these: invalid password, database timeout, `undefined.map()`, missing order, duplicate email, and unexpected circular import.

**Interview answer:** Operational errors are expected runtime conditions that the application can handle safely; programmer errors reveal bugs or broken assumptions. Known errors get meaningful status codes, while unexpected failures get safe 500 responses and are investigated. Process-level failures may require controlled shutdown.
