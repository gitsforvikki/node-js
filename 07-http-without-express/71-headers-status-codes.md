# Lesson 71 — Headers and Status Codes

## Why this lesson matters

A professional API is not only about JSON.

HTTP communicates meaning through:

- status codes
- headers
- body

Interviewers expect you to know which status code fits which scenario and what important headers do.

---

# Part 1 — Status Codes

HTTP status codes are grouped by class.

```text
1xx -> informational
2xx -> success
3xx -> redirection
4xx -> client error
5xx -> server error
```

---

## 1. 200 OK

General successful request.

```http
200 OK
```

Common for:
- successful GET
- successful update with response body

---

## 2. 201 Created

Use when a new resource is successfully created.

```http
POST /users
-> 201 Created
```

Often combined with:

```http
Location: /users/123
```

---

## 3. 202 Accepted

Means:

> request accepted for processing, but work may not be complete yet

Useful for:
- background jobs
- async report generation
- queued processing

Example:

```text
POST /reports
   |
   v
202 Accepted
   |
   v
job continues in background
```

---

## 4. 204 No Content

Successful request with no response body.

Common for:

```http
DELETE /users/123
```

or update endpoints that return nothing.

Do not send a meaningful body with 204.

---

## 5. 301 / 302

Redirection status codes.

```text
301 -> permanent redirect
302 -> temporary/found redirect
```

Modern redirect semantics also involve 307/308 when preserving method/body matters.

---

## 6. 304 Not Modified

Used with caching validators.

It means the client can use its cached representation.

This is different from 204.

---

# Client errors

## 7. 400 Bad Request

Malformed or invalid request syntax/input.

Examples:
- invalid JSON
- malformed query
- invalid request format

---

## 8. 401 Unauthorized

Despite the name, this generally means:

> authentication is required or credentials are invalid

Examples:
- missing token
- expired token
- invalid credentials

---

## 9. 403 Forbidden

Means:

> identity may be known, but permission is denied

Example:

```text
user authenticated
but not admin
```

Difference:

```text
401 -> who are you / auth invalid
403 -> I know who you are, but you're not allowed
```

---

## 10. 404 Not Found

Requested resource does not exist.

---

## 11. 405 Method Not Allowed

Route exists, but method is unsupported.

Example:

```text
GET /users allowed
DELETE /users not allowed
```

---

## 12. 409 Conflict

Useful for state conflicts.

Examples:
- duplicate unique resource
- version conflict
- operation conflicts with current state

Example:

```text
email already registered
```

---

## 13. 422 Unprocessable Content

Request format is valid, but semantic validation fails.

Example:

```json
{
  "email": "not-an-email"
}
```

Teams vary between 400 and 422 for validation. Consistency matters.

---

## 14. 429 Too Many Requests

Used for rate limiting.

Often paired with retry-related headers.

---

# Server errors

## 15. 500 Internal Server Error

Unexpected server-side failure.

Do not expose stack traces to clients.

---

## 16. 502 Bad Gateway

A gateway/proxy received an invalid response from an upstream service.

Common with:
- reverse proxies
- API gateways

---

## 17. 503 Service Unavailable

Service temporarily unavailable.

Examples:
- maintenance
- dependency outage
- overloaded service

---

## 18. 504 Gateway Timeout

Gateway/proxy waited too long for upstream response.

---

# Part 2 — Headers

Headers carry metadata.

---

## 19. Content-Type

Describes body format.

```http
Content-Type: application/json
```

For JSON:

```http
application/json; charset=utf-8
```

---

## 20. Accept

Client tells server which response formats it accepts.

```http
Accept: application/json
```

This is part of content negotiation.

---

## 21. Authorization

Common:

```http
Authorization: Bearer <token>
```

Never log raw bearer tokens in production.

---

## 22. Cookie

Client sends cookies:

```http
Cookie: sessionId=abc123
```

Server sets them using:

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

---

## 23. Cache-Control

Controls caching behavior.

Examples:

```http
Cache-Control: no-store
```

or:

```http
Cache-Control: public, max-age=3600
```

---

## 24. ETag

Used for cache validation.

Server:

```http
ETag: "abc123"
```

Client later:

```http
If-None-Match: "abc123"
```

If unchanged:

```http
304 Not Modified
```

---

## 25. CORS headers

Examples:

```http
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
```

These control browser cross-origin access.

CORS is enforced by browsers, not as a generic server-to-server security boundary.

---

## 26. Location

Common with creation or redirects.

```http
Location: /users/123
```

---

## 27. Retry-After

Useful with:

```text
429
503
```

Example:

```http
Retry-After: 60
```

---

## 28. Security headers

Important examples:

- Strict-Transport-Security
- Content-Security-Policy
- X-Content-Type-Options

Higher-level libraries such as Helmet help configure many of these.

---

## 29. Setting headers in Node.js

```js
res.setHeader(
  "Content-Type",
  "application/json"
);
```

Multiple:

```js
res.writeHead(
  200,
  {
    "Content-Type":
      "application/json",
    "Cache-Control":
      "no-store",
  }
);
```

---

## 30. Status-code decision mental model

Ask:

```text
Did request succeed?
   |
   +--> yes
   |     |
   |     +--> created? -> 201
   |     +--> no body? -> 204
   |     +--> async queued? -> 202
   |     +--> normal -> 200
   |
   +--> no
         |
         +--> bad input? -> 400/422
         +--> not authenticated? -> 401
         +--> not allowed? -> 403
         +--> missing? -> 404
         +--> conflict? -> 409
         +--> rate limit? -> 429
         +--> server failed? -> 500
```

---

## 31. Common mistakes

### Mistake 1
Returning 200 for every response.

### Mistake 2
Using 401 and 403 interchangeably.

### Mistake 3
Returning 500 for validation errors.

### Mistake 4
Leaking stack traces in 500 responses.

### Mistake 5
Ignoring Content-Type.

---

## 32. Interview questions

### 401 vs 403?

401 means authentication is missing/invalid. 403 means authenticated identity lacks permission.

### 400 vs 422?

400 is commonly used for malformed/invalid requests; 422 is often used when syntax is valid but semantic validation fails.

### 200 vs 201?

200 is generic success; 201 specifically indicates resource creation.

### When use 202?

When processing has been accepted but is not complete.

### 204 vs 304?

204 means success with no content. 304 means cached content has not changed.

---

## 33. Strong interview answer

> HTTP status codes should communicate the outcome of the request rather than always returning 200. For example, 201 represents resource creation, 401 authentication failure, 403 permission denial, 404 missing resources, 409 state conflicts, and 429 rate limits. Headers provide metadata such as Content-Type, authorization, caching, cookies, CORS, and retry instructions. Good API design uses both status and headers consistently so clients can understand behavior without guessing from the body alone.

---

## Interview-Ready Summary

```text
2xx success
3xx redirects/cache
4xx client errors
5xx server errors

Must know:
200
201
202
204
400
401
403
404
405
409
422
429
500
502
503
504

Headers:
Content-Type
Accept
Authorization
Cookie
Set-Cookie
Cache-Control
ETag
CORS
Location
Retry-After
```

## Section 7 Progress Map

```text
HTTP Fundamentals
   |
   +--> request/response model
   +--> raw Node HTTP server
   +--> req/res objects
   +--> method semantics
   +--> status + headers
   |
   +--> next:
          query params
          route params
          body parsing
          manual routing
          REST fundamentals
```

## Practice Task

Design responses for these scenarios:

1. invalid JSON
2. user created
3. duplicate email
4. missing auth token
5. authenticated non-admin
6. queued report
7. rate limit exceeded
8. downstream service unavailable

Choose status codes and important headers.
