# Lesson 102 — Error Response Design

## Why error response design matters

Successful API responses get most of the attention.

But in production, clients spend a huge amount of time handling failures.

A good error contract should help clients answer:

- What went wrong?
- Is this my fault or the server's?
- Can I retry?
- Which field is invalid?
- Can support trace the failure?
- Is the message safe to show to a user?

A weak error design makes every client harder to build.

---

## 1. Do not return random error shapes

Bad API:

Endpoint A:

```json
{
  "message": "User not found"
}
```

Endpoint B:

```json
{
  "error": "Duplicate email"
}
```

Endpoint C:

```json
{
  "success": false,
  "reason": "Invalid token"
}
```

This forces every client to write endpoint-specific error parsing.

---

## 2. Use a stable error envelope

Example:

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found",
    "requestId": "req_abc123"
  }
}
```

This gives clients a predictable shape.

---

## 3. HTTP status vs application error code

HTTP status:

```text
404
```

Application error code:

```text
USER_NOT_FOUND
```

Why both?

HTTP status gives broad protocol meaning.

Application code gives domain-specific meaning.

Example:

```text
409 Conflict
```

may represent:

```text
EMAIL_ALREADY_EXISTS
ORDER_ALREADY_CANCELLED
PAYMENT_ALREADY_CAPTURED
```

---

## 4. Stable machine-readable error code

A client should not depend on:

```text
"Email already exists"
```

because human text may change.

Prefer:

```text
EMAIL_ALREADY_EXISTS
```

Use stable codes for programmatic behavior.

---

## 5. Human-readable message

Example:

```json
{
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "Email already registered"
}
```

The message is useful for:
- logs
- debugging
- user-facing fallback

But clients should primarily branch on `code`.

---

## 6. Validation errors

Example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request",
    "details": [
      {
        "field": "email",
        "code": "INVALID_EMAIL",
        "message": "Enter a valid email address"
      },
      {
        "field": "password",
        "code": "TOO_SHORT",
        "message": "Password must be at least 8 characters"
      }
    ]
  }
}
```

This is much easier for frontends to map into form errors.

---

## 7. Field path format

For nested objects:

```text
address.city
items[0].quantity
```

Pick one path format and use it consistently.

Examples:
- dot notation
- JSON Pointer
- array of path segments

---

## 8. Request IDs

Include or expose a request/correlation ID.

Example header:

```http
X-Request-Id: req_abc123
```

Optional body:

```json
{
  "error": {
    "requestId": "req_abc123"
  }
}
```

This lets support correlate a client failure with server logs.

---

## 9. Retryability

Some errors are retryable.

Example:

```text
503 Service Unavailable
429 Too Many Requests
504 Gateway Timeout
```

Your response may include:

```http
Retry-After: 60
```

Do not invent a custom JSON-only retry mechanism when HTTP already has relevant headers.

---

## 10. Do not expose stack traces

Bad:

```json
{
  "error": {
    "message": "TypeError...",
    "stack": "at /app/src/..."
  }
}
```

This can leak:
- file paths
- package versions
- internals
- DB details
- implementation assumptions

Stack traces belong in logs, not public responses.

---

## 11. Do not expose DB errors

Bad:

```json
{
  "error": "duplicate key value violates unique constraint users_email_key"
}
```

Better:

```json
{
  "error": {
    "code": "EMAIL_ALREADY_EXISTS",
    "message": "Email already registered"
  }
}
```

Translate infrastructure errors to domain-level errors.

---

## 12. Operational vs programmer errors

Operational errors:

- validation failure
- not found
- conflict
- unauthorized
- rate limited

Programmer errors:

- null dereference
- impossible state
- coding bug

Public error handling should not treat both as if they are safe to expose.

---

## 13. Authentication errors

Example 401:

```json
{
  "error": {
    "code": "AUTH_REQUIRED",
    "message": "Authentication required"
  }
}
```

Avoid leaking whether:
- email exists
- password was wrong
- token subject exists

when that detail creates security risk.

---

## 14. Login response privacy

Bad:

```text
"No account exists for this email"
```

This may enable user enumeration.

Safer:

```text
"Invalid credentials"
```

Security sometimes requires less-specific messages.

---

## 15. 404 vs 403 for hidden resources

Sometimes:

```text
403
```

reveals that a protected resource exists.

For sensitive resources, returning:

```text
404
```

may reduce information disclosure.

This is a deliberate API/security decision.

---

## 16. Error mapping layer

A healthy architecture:

```text
Database error
      |
      v
Repository
      |
      v
Domain/Application Error
      |
      v
Error Middleware
      |
      v
HTTP Status + Error Contract
```

---

## 17. Custom error class

Example:

```js
class AppError extends Error {
  constructor({
    code,
    message,
    statusCode = 500,
    details,
  }) {
    super(message);

    this.code = code;
    this.statusCode = statusCode;
    this.details = details;
  }
}
```

---

## 18. Example mapping

```js
throw new AppError({
  code:
    "ORDER_NOT_FOUND",
  message:
    "Order not found",
  statusCode: 404,
});
```

Central middleware formats it.

---

## 19. Logging context

Useful error log fields:

```text
requestId
userId
method
path
errorCode
stack
timestamp
dependency
latency
```

Do not log:
- passwords
- raw tokens
- card numbers
- secrets

---

## 20. Consistency across services

In microservices, a shared error shape helps gateways and clients.

Example:

```text
auth service
orders service
payments service
```

all return the same broad error contract.

---

## 21. Error chaining

When wrapping errors, preserve the root cause in logs where possible.

Conceptually:

```text
DB timeout
   |
   v
PaymentRepositoryError
   |
   v
PaymentUnavailableError
```

Public response may stay simple while internal diagnostics retain cause.

---

## 22. Do not overexpose validation internals

A validator library may return large internal structures.

Map them into your own API format.

This protects your contract from library-specific details.

---

## 23. Rate limit error

Example:

```http
429 Too Many Requests
Retry-After: 60
```

Body:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests"
  }
}
```

---

## 24. Service unavailable error

Example:

```http
503 Service Unavailable
```

Body:

```json
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Service temporarily unavailable",
    "requestId": "req_123"
  }
}
```

---

## 25. Common mistakes

### Mistake 1
Different error shapes per endpoint.

### Mistake 2
Using human messages as machine codes.

### Mistake 3
Leaking stack traces.

### Mistake 4
Returning raw DB/provider errors.

### Mistake 5
No request ID.

### Mistake 6
Exposing security-sensitive details.

---

## 26. Interview questions

### Why use application error codes?

To let clients handle specific domain failures consistently while HTTP status communicates broad protocol meaning.

### Should you expose stack traces?

No.

### What should validation errors include?

A stable top-level error code and structured field-level details.

### Why include requestId?

To correlate client failures with server logs and traces.

---

## 27. Strong interview answer

> I design one consistent error contract for the whole API. The HTTP status communicates the broad failure class, while a stable application error code communicates the exact domain condition. Validation errors include structured field details, internal infrastructure errors are translated before leaving the server, and every unexpected error is logged with a request ID. I never expose stack traces, raw database errors, tokens, or other sensitive internals.

---

## Interview-Ready Summary

```text
HTTP Status
   +
Application Error Code
   +
Safe Message
   +
Optional Details
   +
Request ID

Never expose:
stack traces
DB errors
tokens
secrets
internal implementation
```

## Practice Task

Design error responses for:

1. invalid email
2. duplicate email
3. missing JWT
4. forbidden admin action
5. hidden private resource
6. DB timeout
7. rate limit exceeded
8. payment provider unavailable
