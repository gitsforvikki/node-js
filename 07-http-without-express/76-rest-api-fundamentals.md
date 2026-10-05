# Lesson 76 — REST API Fundamentals

## Why this lesson matters

REST is one of the most common API design styles in backend interviews.

A strong answer is not:

> REST means using GET, POST, PUT, DELETE.

That is incomplete.

REST is an architectural style centered around resources, representations, stateless communication, and standardized HTTP semantics.

---

## 1. What does REST mean?

REST stands for:

```text
Representational State Transfer
```

It is an architectural style introduced for networked systems.

In practical web APIs, REST commonly means modeling the system around resources and using HTTP semantics consistently.

---

## 2. Think in resources, not actions

Action-style:

```text
POST /createUser
POST /getUser
POST /deleteUser
```

Resource-oriented:

```text
POST   /users
GET    /users/:id
DELETE /users/:id
```

The resource is:

```text
users
```

The HTTP method describes the action.

---

## 3. Resource representation

The server resource may be stored in a database.

The API sends a representation of that resource.

Example:

```json
{
  "id": 123,
  "name": "Vikash"
}
```

This is a representation, not necessarily the exact database object.

---

## 4. Common REST routes

```text
GET    /users
GET    /users/:id
POST   /users
PUT    /users/:id
PATCH  /users/:id
DELETE /users/:id
```

---

## 5. Statelessness

REST emphasizes stateless requests.

Each request should contain enough information for the server to understand it.

Example:

```http
Authorization: Bearer token
```

The server should not require hidden per-request conversational state.

---

## 6. Stateless does not mean no server-side data

This is a common misunderstanding.

You can absolutely have:
- databases
- sessions
- caches

Statelessness means each request can be understood independently at the protocol interaction level.

---

## 7. Uniform interface

REST encourages consistent resource access through standard HTTP semantics.

That includes:
- methods
- URIs
- status codes
- representations
- headers

This reduces custom protocol behavior.

---

## 8. Client-server separation

Client and server evolve independently.

```text
Frontend
   |
   v
HTTP API
   |
   v
Backend
```

The frontend should not need to know:
- database schema
- internal services
- storage engine

---

## 9. Cacheability

Responses can communicate caching rules.

Example:

```http
Cache-Control: public, max-age=60
```

Caching can improve:
- latency
- bandwidth
- backend load

---

## 10. Layered system

A client may talk to:

```text
Client
   |
   v
CDN
   |
   v
Load Balancer
   |
   v
API Gateway
   |
   v
Node.js API
```

The client does not need to know every internal layer.

---

## 11. Idempotency in REST

Important method semantics:

```text
GET    -> idempotent
PUT    -> idempotent
DELETE -> idempotent

POST   -> generally not idempotent
PATCH  -> depends on operation semantics
```

This matters for retries.

---

## 12. Resource naming

Prefer nouns:

Good:

```text
/users
/orders
/products
```

Avoid unnecessary verbs:

```text
/getUsers
/createOrder
/deleteProduct
```

---

## 13. Nested resources

Useful:

```text
/users/123/orders
```

But avoid excessive nesting.

Often:

```text
/orders/456
```

is enough once the resource has its own identity.

---

## 14. Filtering collections

Use query parameters:

```text
GET /products?category=laptop
```

Not:

```text
GET /products/category/laptop/search
```

unless the resource model specifically justifies it.

---

## 15. Pagination

Example:

```text
GET /users?page=2&limit=20
```

Production APIs should avoid returning unlimited collections.

Other pagination styles include:
- offset pagination
- cursor pagination

These are covered more deeply later.

---

## 16. Error responses

A good REST API should return consistent errors.

Example:

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found"
  }
}
```

Status:

```text
404
```

Do not use HTTP 200 for every error.

---

## 17. Validation

Request:

```json
{
  "email": "wrong"
}
```

A REST API should:
- validate
- return client error
- explain safely

Example:

```text
400 or 422
```

depending on API convention.

---

## 18. Versioning

Common strategies:

```text
/api/v1/users
```

or header-based versioning.

Do not version every tiny change.

Version when compatibility boundaries require it.

---

## 19. REST maturity

Not every API called "REST" follows every theoretical REST constraint.

In industry, "REST API" often means:
- resource-oriented
- HTTP-based
- JSON representations
- method/status semantics
- stateless request design

Be precise without becoming dogmatic in interviews.

---

## 20. REST vs RPC

REST-style:

```text
POST /orders
```

RPC-style:

```text
POST /createOrder
```

Neither is universally superior.

REST is strong when resource modeling fits well.

RPC can be appropriate for command-heavy/domain-specific operations.

---

## 21. REST vs GraphQL

REST:

```text
multiple resource endpoints
server defines representations
```

GraphQL:

```text
typically one graph endpoint
client chooses requested fields
```

Different trade-offs.

---

## 22. Real API example

Create order:

```http
POST /orders
Content-Type: application/json
Authorization: Bearer token
```

Response:

```text
201 Created
Location: /orders/987
```

Body:

```json
{
  "id": 987,
  "status": "pending"
}
```

This combines:
- resource URI
- HTTP method
- status
- representation
- headers

---

## 23. Common mistakes

### Mistake 1
Calling every HTTP JSON API REST.

### Mistake 2
Using verbs in every route.

### Mistake 3
Ignoring HTTP status semantics.

### Mistake 4
Returning unlimited collections.

### Mistake 5
Putting business state in hidden client-server conversations.

### Mistake 6
Thinking REST means CRUD only.

REST can represent workflows too, as long as resources are modeled sensibly.

---

## 24. Interview questions

### What is REST?

An architectural style for distributed systems that emphasizes resources, representations, stateless communication, uniform interfaces, cacheability, and layered systems.

### Is REST the same as CRUD?

No. CRUD maps naturally to many REST operations, but REST is a broader architectural style.

### Why use nouns in URLs?

Because URLs identify resources while HTTP methods communicate the operation.

### What does stateless mean?

Each request contains the information necessary to understand/process it without depending on hidden conversational state from previous requests.

### Is every JSON-over-HTTP API REST?

No.

---

## 25. Strong interview answer

> REST is an architectural style centered around resources and representations. In practical APIs, I model URLs as resources such as /users or /orders, use HTTP methods to express operations, return meaningful status codes, keep requests stateless, support caching where appropriate, and expose consistent representations. REST is broader than CRUD and should not be reduced to simply using GET, POST, PUT, and DELETE.

---

## Interview-Ready Summary

```text
REST
   |
   +--> resources
   +--> representations
   +--> stateless requests
   +--> uniform HTTP interface
   +--> cacheability
   +--> layered systems

Good URLs:
  /users
  /orders/:id

Methods describe action.
Resources stay noun-oriented.
```

## Section 7 Final Mental Model

```text
Client Request
   |
   v
HTTP Method + URL
   |
   +--> Route Param
   +--> Query Params
   +--> Headers
   +--> Body
   |
   v
Raw Node Router
   |
   v
Validation
   |
   v
Business Logic
   |
   v
HTTP Status + Headers + Body
   |
   v
Client Response
```

If you can build and explain this flow without Express, you understand what Express is actually simplifying.

## Practice Task

Build a complete raw Node.js REST API for:

```text
/users
```

Support:

- GET /users
- GET /users/:id
- POST /users
- PATCH /users/:id
- DELETE /users/:id
- pagination
- validation
- correct status codes
- JSON body parsing
- 404 and 405
