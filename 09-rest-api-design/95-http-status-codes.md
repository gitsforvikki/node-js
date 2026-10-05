# Lesson 95 — HTTP Status Codes

## Why this lesson matters

Status codes are part of your API contract.

A professional API should not force clients to inspect a JSON message just to understand whether a request succeeded.

Correct status codes improve:

- client behavior
- retries
- caching
- observability
- monitoring
- debugging

---

# Status code families

```text
1xx -> Informational
2xx -> Success
3xx -> Redirection / cache
4xx -> Client-side request problem
5xx -> Server-side failure
```

---

# Success Codes

## 1. 200 OK

General successful request.

Use for:
- GET success
- update with response body
- action returning result

---

## 2. 201 Created

A new resource was created.

Example:

```text
POST /users
-> 201
```

Often include:

```http
Location: /users/123
```

---

## 3. 202 Accepted

The request was accepted but processing is not complete.

Example:

```text
POST /reports
   |
   v
queue job
   |
   v
202 Accepted
```

Useful for async/background work.

---

## 4. 204 No Content

Successful operation with no response body.

Example:

```text
DELETE /users/123
```

Do not send a normal response body with 204.

---

# Redirection and cache

## 5. 301 Moved Permanently

Permanent redirect.

---

## 6. 302 Found

Temporary redirect behavior.

For method-preserving redirects, 307/308 are often more precise.

---

## 7. 304 Not Modified

Used for conditional caching.

Example:

```http
If-None-Match: "abc"
```

If resource unchanged:

```http
304 Not Modified
```

No normal response body is needed.

---

# Client Errors

## 8. 400 Bad Request

Use for malformed or invalid request syntax/input.

Examples:
- malformed JSON
- invalid query syntax
- invalid basic input shape

---

## 9. 401 Unauthorized

Despite the name, this means:

> authentication is required or credentials are invalid.

Examples:
- missing token
- invalid token
- expired token

---

## 10. 403 Forbidden

The caller is authenticated but not permitted.

Example:

```text
user role = user
required role = admin
```

---

## 11. 404 Not Found

The requested route/resource does not exist.

---

## 12. 405 Method Not Allowed

The resource path exists but the HTTP method is unsupported.

Example:

```text
GET /users supported
DELETE /users unsupported
```

---

## 13. 409 Conflict

Use when request conflicts with current resource state.

Examples:
- duplicate unique account
- optimistic concurrency conflict
- invalid state transition

---

## 14. 410 Gone

Resource existed before but is intentionally no longer available.

Useful in some deprecation/deletion scenarios.

---

## 15. 412 Precondition Failed

Useful with conditional updates.

Example:

```http
If-Match: "version-5"
```

If version changed:

```text
412
```

This can support optimistic concurrency.

---

## 16. 413 Content Too Large

Request payload exceeds allowed size.

Example:
- JSON body too large
- upload too large

---

## 17. 415 Unsupported Media Type

Example:

Client sends:

```http
Content-Type: text/plain
```

but endpoint requires JSON.

---

## 18. 422 Unprocessable Content

Often used when syntax is valid but semantic validation fails.

Example:

```json
{
  "email": "invalid"
}
```

Some teams use 400 instead.

Consistency matters more than endless debate.

---

## 19. 429 Too Many Requests

Rate limit exceeded.

Often combine with:

```http
Retry-After: 60
```

---

# Server Errors

## 20. 500 Internal Server Error

Unexpected server failure.

Never expose raw stack traces.

---

## 21. 501 Not Implemented

Server does not support required functionality.

Less common in normal CRUD APIs.

---

## 22. 502 Bad Gateway

Gateway/proxy received a bad response from upstream.

Example:

```text
API Gateway
   |
   v
downstream service returned invalid response
```

---

## 23. 503 Service Unavailable

Service is temporarily unavailable.

Examples:
- maintenance
- overloaded dependency
- readiness failure

May include:

```http
Retry-After
```

---

## 24. 504 Gateway Timeout

Gateway/proxy did not receive upstream response in time.

---

# Authentication Status Decision

## 25. 401 vs 403

Mental model:

```text
Do I know who you are?
   |
   +--> no -> 401
   |
   +--> yes
          |
          v
Are you allowed?
   |
   +--> no -> 403
```

---

# Validation Status Decision

## 26. 400 vs 422

Possible convention:

```text
400
  malformed syntax/basic request

422
  syntactically valid but semantically invalid
```

Example:

Malformed JSON:

```text
400
```

Valid JSON with invalid email:

```text
422
```

---

## 27. 404 and security

Sometimes APIs intentionally return 404 instead of 403 for resources the caller should not know exist.

Example:

```text
GET /private-documents/123
```

This can reduce information disclosure.

---

## 28. Status code and retry behavior

Client should generally not retry every error.

Example:

```text
400
  retrying same request usually useless

429
  retry later

503
  retry may succeed

504
  retry may succeed, but idempotency matters
```

Status codes influence resilient client behavior.

---

## 29. Consistent error body

Example:

```json
{
  "error": {
    "code": "EMAIL_ALREADY_EXISTS",
    "message": "Email already registered",
    "requestId": "abc-123"
  }
}
```

HTTP status communicates broad category.

Application error code gives precise meaning.

---

## 30. Do not return 200 for errors

Bad:

```http
200 OK

{
  "success": false,
  "message": "User not found"
}
```

This breaks:
- monitoring
- caches
- client logic
- observability

Use:

```text
404
```

---

## 31. Common mistakes

### Mistake 1
Using 500 for client validation errors.

### Mistake 2
Using 401 and 403 interchangeably.

### Mistake 3
Returning 200 for everything.

### Mistake 4
Sending body with 204.

### Mistake 5
Ignoring retry semantics.

---

## 32. Interview questions

### 200 vs 201?

200 is generic success; 201 indicates resource creation.

### 401 vs 403?

401 is missing/invalid authentication; 403 is insufficient permission.

### 400 vs 422?

400 is commonly malformed/basic invalid input; 422 is often semantically invalid input.

### 409 vs 422?

409 represents conflict with current resource state; 422 represents invalid input semantics.

### 502 vs 503 vs 504?

502 bad upstream response, 503 temporary service unavailability, 504 upstream timeout.

---

## 33. Strong interview answer

> HTTP status codes should communicate the outcome of the request at the protocol level. I use 2xx for success, 4xx when the client request cannot be fulfilled as sent, and 5xx for server-side failures. I distinguish 401 from 403, use 409 for state conflicts, 429 for rate limits, and avoid returning 200 with an error object. I also consider retry and caching semantics when choosing codes.

---

## Interview-Ready Summary

```text
Success:
200 201 202 204

Client:
400 401 403 404 405
409 412 413 415 422 429

Server:
500 502 503 504

Key comparisons:
401 vs 403
400 vs 422
409 vs 422
502 vs 503 vs 504
```

## Practice Task

Choose status codes for:

1. invalid JSON
2. duplicate email
3. expired JWT
4. authenticated user accessing admin route
5. missing order
6. rate limit exceeded
7. request body too large
8. payment provider timeout
9. queued report generation
10. successful delete with no response body
