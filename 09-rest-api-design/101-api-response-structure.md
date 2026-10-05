# Lesson 101 — Standard API Response Structure

## Why response consistency matters

A good API should not return a completely different shape from every endpoint.

Consistency helps:

- frontend developers
- mobile clients
- error handling
- TypeScript typing
- logging
- API documentation
- testing

But consistency does **not** mean wrapping every response in unnecessary boilerplate.

---

## 1. Basic success response

Simple endpoint:

```json
{
  "id": 123,
  "name": "Vikash"
}
```

This is perfectly valid.

You do not always need:

```json
{
  "success": true,
  "status": 200,
  "data": {
    "id": 123
  }
}
```

if HTTP already communicates those values.

---

## 2. Envelope pattern

Some APIs standardize:

```json
{
  "data": {
    "id": 123,
    "name": "Vikash"
  }
}
```

Benefits:
- room for metadata
- predictable shape

---

## 3. Collection response

```json
{
  "data": [
    {
      "id": 1
    },
    {
      "id": 2
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 200
  }
}
```

This is a good reason to use an envelope.

---

## 4. Cursor response

```json
{
  "data": [],
  "pagination": {
    "nextCursor": "abc123",
    "hasMore": true
  }
}
```

---

## 5. Metadata

Possible metadata:

```json
{
  "data": [],
  "meta": {
    "requestId": "abc-123"
  }
}
```

Avoid stuffing arbitrary unrelated metadata into every response.

---

## 6. Error response

A predictable error shape is especially important.

Example:

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found",
    "requestId": "abc-123"
  }
}
```

---

## 7. Application error code

HTTP:

```text
404
```

Application code:

```text
USER_NOT_FOUND
```

Why both?

HTTP status:
- broad protocol meaning

Application code:
- precise business meaning

---

## 8. Validation errors

Example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request",
    "details": [
      {
        "field": "email",
        "message": "Invalid email"
      }
    ]
  }
}
```

---

## 9. Do not expose internal details

Bad:

```json
{
  "error": "MongoServerError E11000 collection users index email_1..."
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

---

## 10. Status code belongs in HTTP

Avoid relying on:

```json
{
  "statusCode": 404
}
```

while returning:

```text
200 OK
```

Use actual HTTP status:

```text
404
```

---

## 11. Avoid redundant success flag

Example:

```json
{
  "success": true
}
```

is often redundant if response status is 2xx.

It is not wrong, but ask whether it adds real value.

---

## 12. Null vs missing fields

Be consistent.

Example:

```json
{
  "middleName": null
}
```

vs omitted field:

```json
{}
```

These may mean different things.

Document your contract.

---

## 13. Date format

Use consistent standardized timestamps.

Common choice:

```text
ISO 8601 / RFC 3339 style
```

Example:

```json
{
  "createdAt": "2026-10-06T10:30:00Z"
}
```

Avoid locale-specific ambiguous dates.

---

## 14. Monetary values

Do not casually use floating-point values for money.

Potential strategies:

```json
{
  "amount": 199900,
  "currency": "INR"
}
```

where amount is paise.

Or use precise decimal representation according to contract.

Always include currency when relevant.

---

## 15. IDs

Keep ID representation consistent.

If IDs are strings:

```json
{
  "id": "usr_123"
}
```

do not sometimes return numbers and sometimes strings.

---

## 16. Response serializers

Do not expose raw DB objects directly.

Example:

```js
function serializeUser(
  user
) {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
  };
}
```

This prevents leaking:
- password hashes
- internal flags
- DB metadata

---

## 17. Version-specific serializers

```text
Service
   |
   v
domain object
   |
   +--> v1 serializer
   |
   +--> v2 serializer
```

This is cleaner than versioning business logic.

---

## 18. Partial responses

If API supports field selection:

```text
?fields=id,name
```

make contract clear.

Do not accidentally omit required identifiers/links if clients depend on them.

---

## 19. Empty collection

Prefer:

```json
{
  "data": []
}
```

not:

```json
{
  "data": null
}
```

for a collection with no records.

---

## 20. Single missing resource

If resource does not exist:

```text
404
```

Do not return:

```json
{
  "data": null
}
```

with 200 unless your API contract intentionally defines that behavior.

---

## 21. Creation response

Example:

```text
201 Created
Location: /users/123
```

Body:

```json
{
  "data": {
    "id": 123,
    "name": "Vikash"
  }
}
```

---

## 22. Delete response

Option:

```text
204 No Content
```

Then send no body.

Another valid design:

```text
200
```

with deleted resource/result.

Pick one convention.

---

## 23. Async operation response

Example:

```text
202 Accepted
```

Body:

```json
{
  "data": {
    "jobId": "job_123",
    "status": "queued"
  }
}
```

Potential Location:

```text
/jobs/job_123
```

---

## 24. Response headers are part of the contract

Examples:

- Location
- ETag
- Cache-Control
- Retry-After
- RateLimit-related headers
- Content-Type

Do not put everything into JSON if HTTP already has a standard header.

---

## 25. Request IDs

For debugging:

```http
X-Request-Id: abc-123
```

and optionally:

```json
{
  "error": {
    "requestId": "abc-123"
  }
}
```

This helps correlate client reports with logs.

---

## 26. Do not leak authorization data

Serializer should not expose:

```text
roles
permissions
internal flags
```

unless the client genuinely needs them.

---

## 27. Response consistency without overengineering

Good principle:

> Standardize where consistency helps clients, but do not wrap every scalar response in five layers of metadata.

---

## 28. Common mistakes

### Mistake 1
200 status for every result.

### Mistake 2
Raw DB objects returned directly.

### Mistake 3
Different error format per endpoint.

### Mistake 4
Redundant status code only in JSON.

### Mistake 5
Inconsistent ID/date/money formats.

### Mistake 6
Returning null instead of [] for empty collections.

---

## 29. Interview questions

### Should every API response use `{ success, data }`?

Not necessarily. HTTP status already communicates success; wrappers are useful mainly when they provide consistent metadata or structure.

### Why use application error codes?

To distinguish precise business failures within the broader HTTP status category.

### Why serialize DB entities?

To protect the API contract and avoid leaking internal fields.

### What should an empty collection return?

Usually an empty array.

---

## 30. Strong interview answer

> I keep API responses consistent but avoid redundant wrappers. For collections, I usually return a data array plus pagination metadata. For errors, I use a stable shape with an application error code, safe message, and request ID. I rely on the actual HTTP status code rather than embedding success/failure only in JSON, and I serialize domain objects instead of returning raw database models so internal fields never leak accidentally.

---

## Interview-Ready Summary

```text
Success:
HTTP status
+
data

Collection:
data[]
+
pagination/meta

Error:
error.code
error.message
requestId

Keep consistent:
IDs
dates
money
null/empty semantics

Never expose raw DB objects.
```

## Section 9 Progress Map

```text
REST Architecture
      |
      v
Resource Design
      |
      v
Methods + Idempotency
      |
      v
Status Codes
      |
      v
Validation
      |
      v
Pagination
      |
      v
Filtering
      |
      v
Sorting
      |
      v
Searching
      |
      v
Response Contract
      |
      v
Next:
Error Response Design
API Versioning
Rate Limiting
OpenAPI / Swagger
```

## Practice Task

Define standard response contracts for:

1. GET /users/:id
2. GET /users?page=2
3. POST /users
4. DELETE /users/:id
5. validation failure
6. duplicate email
7. async report generation
