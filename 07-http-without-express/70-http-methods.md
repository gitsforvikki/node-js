# Lesson 70 — HTTP Methods

## Why HTTP methods matter

HTTP methods communicate the **intent** of a request.

Good API design uses methods consistently.

Main methods:

```text
GET
POST
PUT
PATCH
DELETE
HEAD
OPTIONS
```

---

## 1. GET

Used to retrieve a resource.

Example:

```http
GET /users/123
```

Expected behavior:
- read data
- should not create/update server state as its intended semantic effect

---

## 2. POST

Commonly used to create a resource or trigger a non-idempotent operation.

Example:

```http
POST /users
```

Body:

```json
{
  "name": "Vikash"
}
```

Possible response:

```text
201 Created
```

---

## 3. PUT

Typically means:

> replace the resource representation at this URI

Example:

```http
PUT /users/123
```

Body may contain the full desired representation.

---

## 4. PATCH

Used for partial modification.

Example:

```http
PATCH /users/123
```

Body:

```json
{
  "name": "New Name"
}
```

---

## 5. DELETE

Used to remove a resource.

```http
DELETE /users/123
```

Possible responses:
- 204 No Content
- 200 with result body

---

## 6. HEAD

HEAD is similar to GET but response body is omitted.

Useful for:
- metadata
- existence checks
- content length
- caching logic

---

## 7. OPTIONS

Used to describe communication options for a resource/server.

Very common in CORS preflight requests.

Example:

```http
OPTIONS /users
```

---

# Safety and Idempotency

These terms are heavily tested.

---

## 8. Safe methods

A safe method is intended not to change server state.

Examples:

```text
GET
HEAD
OPTIONS
```

Safe does not mean:
- no logging
- no analytics
- no cache updates

It means the requested semantic action is read-only.

---

## 9. Idempotent methods

An operation is idempotent if repeating the same request has the same intended effect as doing it once.

Examples typically considered idempotent:

```text
GET
PUT
DELETE
HEAD
OPTIONS
```

POST is generally not assumed idempotent.

PATCH may or may not be idempotent depending on semantics.

---

## 10. Idempotency example

DELETE:

```http
DELETE /users/123
```

First call:
- user deleted

Second call:
- user already absent

The resulting resource state is still:

```text
user does not exist
```

That is why DELETE is idempotent semantically.

---

## 11. POST payment problem

Suppose:

```http
POST /payments
```

Client sends request.

Payment succeeds, but response is lost.

Client retries.

Now two payments may be created.

Solution for critical POST operations:

```text
Idempotency-Key
```

The server recognizes retries of the same logical operation.

---

## 12. PUT vs PATCH

```text
PUT
  -> complete replacement semantics

PATCH
  -> partial modification
```

Example user:

```json
{
  "name": "Vikash",
  "city": "Bengaluru"
}
```

PATCH:

```json
{
  "city": "Pune"
}
```

Only city changes.

---

## 13. Method handling in raw Node.js

```js
if (
  req.method === "GET"
) {
  // ...
}

if (
  req.method === "POST"
) {
  // ...
}
```

---

## 14. 405 Method Not Allowed

Suppose route exists:

```text
/users
```

but only supports GET.

If client sends:

```http
DELETE /users
```

a better response may be:

```text
405 Method Not Allowed
```

instead of 404.

This distinction matters.

---

## 15. Allow header

With 405 responses, servers may include:

```http
Allow: GET, POST
```

This tells the client which methods are supported.

---

## 16. REST mapping

Typical REST conventions:

```text
GET    /users
GET    /users/:id
POST   /users
PUT    /users/:id
PATCH  /users/:id
DELETE /users/:id
```

---

## 17. Common mistakes

### Mistake 1
Using GET to perform deletes or writes.

Bad API semantics.

### Mistake 2
Using POST for every endpoint.

Loses meaningful HTTP semantics.

### Mistake 3
Confusing PUT and PATCH.

### Mistake 4
Saying DELETE is not idempotent because second request returns 404.

Idempotency is about intended effect/state, not identical response code.

---

## 18. Interview questions

### GET vs POST?

GET retrieves resources and is safe/idempotent by semantic definition. POST commonly creates resources or performs non-idempotent actions.

### PUT vs PATCH?

PUT generally replaces the full target representation; PATCH applies partial changes.

### What is idempotency?

Repeating the same request has the same intended effect as executing it once.

### Is DELETE idempotent?

Yes, semantically.

---

## 19. Strong interview answer

> HTTP methods communicate operation semantics. GET retrieves resources and should be safe. POST is usually used for creation or non-idempotent actions. PUT represents full replacement semantics, PATCH partial modification, and DELETE removes a resource. Idempotency is important because retries happen in distributed systems; GET, PUT, and DELETE are semantically idempotent, while POST usually needs an explicit idempotency strategy for critical writes such as payments.

---

## Interview-Ready Summary

```text
GET
  -> read

POST
  -> create/action

PUT
  -> full replace

PATCH
  -> partial update

DELETE
  -> remove

HEAD
  -> GET metadata, no body

OPTIONS
  -> supported communication options

Interview:
safe vs idempotent
```

## Practice Task

Design method + endpoint combinations for:

- users
- orders
- profile update
- password reset
- payment creation
- payment retry

Explain idempotency for each.
