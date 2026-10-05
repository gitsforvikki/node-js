# Lesson 92 — REST Architecture

## Why this lesson matters

REST is one of the most commonly discussed backend interview topics.

A weak answer is:

> REST means using GET, POST, PUT, PATCH, and DELETE.

That is incomplete.

REST is an **architectural style** with constraints that shape how distributed systems communicate.

If you understand the architecture, you can explain:

- why resources matter
- why requests should be stateless
- why caching matters
- why HTTP semantics matter
- why REST is broader than CRUD
- what makes an API RESTful vs merely JSON-over-HTTP

---

## 1. What does REST stand for?

REST stands for:

```text
Representational State Transfer
```

The term was introduced by Roy Fielding.

REST describes a set of architectural constraints for networked systems.

---

## 2. Resource-centered design

REST models a system around **resources**.

Examples:

```text
/users
/orders
/products
/payments
/subscriptions
```

A resource is some identifiable thing in the system.

---

## 3. Representation

The client usually does not receive the database record directly.

It receives a **representation** of the resource.

Example:

```json
{
  "id": 123,
  "name": "Vikash Kumar"
}
```

This is one representation of the user resource.

The actual internal model may contain:
- password hash
- internal flags
- DB metadata
- audit fields

Those should not automatically be exposed.

---

## 4. REST constraints

The core REST constraints are commonly described as:

```text
Client-Server
Stateless
Cacheable
Uniform Interface
Layered System
Code on Demand (optional)
```

For practical backend interviews, the first five matter most.

---

# Client-Server

## 5. Client-server separation

The client and server should evolve independently.

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
- DB schema
- ORM details
- internal service topology

The server should not depend on one specific frontend implementation.

---

# Statelessness

## 6. What stateless means

Each request should contain enough information to be understood independently.

Example:

```http
Authorization: Bearer <token>
```

The server should not require hidden conversational state from a previous request.

---

## 7. Stateless does not mean no state exists

This is a common interview trap.

A REST API can absolutely use:

- databases
- Redis
- sessions
- caches
- queues

Statelessness means:

> request processing should not depend on implicit client-session conversation state stored between requests.

---

# Cacheability

## 8. Cacheable responses

Responses can declare whether they are cacheable.

Example:

```http
Cache-Control: public, max-age=60
```

Benefits:

- lower latency
- fewer DB calls
- reduced server load
- lower bandwidth

---

## 9. Validation caching

HTTP supports conditional requests.

Example:

Server:

```http
ETag: "abc123"
```

Client:

```http
If-None-Match: "abc123"
```

If unchanged:

```http
304 Not Modified
```

---

# Uniform Interface

## 10. Uniform interface

Clients interact with resources consistently through:

- URLs
- HTTP methods
- status codes
- headers
- representations

Example:

```text
GET    /users/123
PATCH  /users/123
DELETE /users/123
```

The resource stays the same.

The method expresses the operation.

---

## 11. Why uniformity matters

Without consistent conventions:

```text
/getUser
/update-user
/removeUserById
/delete_user
```

every endpoint becomes custom.

Uniform design reduces cognitive load.

---

# Layered System

## 12. Layered architecture

The client may not know whether it is talking directly to the application server.

Example:

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
Node.js Service
```

Each layer can have a responsibility.

---

## 13. Why layering matters

Layers support:

- caching
- security
- routing
- rate limiting
- load balancing
- observability

The client should not need to understand every layer.

---

# Code on Demand

## 14. Optional constraint

REST also describes an optional "code on demand" constraint where servers can send executable code to clients.

This is rarely the main focus in API interviews.

---

## 15. REST vs CRUD

CRUD:

```text
Create
Read
Update
Delete
```

REST is broader.

CRUD describes data operations.

REST describes an architectural interaction style.

---

## 16. Example mapping

```text
Create
  -> POST /users

Read
  -> GET /users/:id

Update
  -> PATCH /users/:id

Delete
  -> DELETE /users/:id
```

Useful mapping, but not the definition of REST.

---

## 17. REST vs RPC

REST:

```text
POST /orders
```

RPC:

```text
POST /createOrder
```

RPC focuses on actions/functions.

REST focuses on resources and state transitions.

Neither is always better.

---

## 18. REST vs GraphQL

REST:

```text
multiple resource endpoints
server-defined response shapes
HTTP semantics
```

GraphQL:

```text
usually one endpoint
client-defined field selection
graph-oriented schema
```

They solve different problems.

---

## 19. HATEOAS

A stricter REST interpretation includes hypermedia links that guide clients through available actions.

Example:

```json
{
  "id": 123,
  "status": "pending",
  "links": {
    "self": "/orders/123",
    "cancel": "/orders/123/cancellation"
  }
}
```

Many industry APIs are called RESTful even if they do not fully implement HATEOAS.

Be precise in interviews.

---

## 20. Statelessness and authentication

JWT-style example:

```text
Request
   |
   +--> Authorization token
   |
   v
Server can authenticate independently
```

Session-based auth can also coexist with REST-style APIs, but the architecture should minimize hidden conversational coupling.

---

## 21. Caching and unsafe data

Do not cache sensitive responses carelessly.

Example:

```text
GET /profile
```

may need:

```http
Cache-Control: no-store
```

while:

```text
GET /public/products
```

may be cacheable.

---

## 22. Resource identity

Good REST URLs identify nouns:

```text
/users/123
/orders/456
/products/789
```

Avoid mixing storage implementation into URLs.

Bad:

```text
/mongodb/users/123
```

Clients should not care how data is stored.

---

## 23. REST and state transitions

REST is not limited to simple CRUD.

Example order workflow:

```text
pending
   |
   v
paid
   |
   v
shipped
```

You can model transitions as resources or carefully designed operations.

---

## 24. Common mistakes

### Mistake 1
REST = JSON over HTTP.

Wrong.

### Mistake 2
REST = CRUD only.

Wrong.

### Mistake 3
Stateless = server stores no data.

Wrong.

### Mistake 4
Ignoring caching entirely.

### Mistake 5
Using inconsistent HTTP semantics.

---

## 25. Interview questions

### What is REST?

An architectural style for distributed systems built around constraints such as client-server separation, statelessness, cacheability, uniform interfaces, and layered systems.

### What is a resource?

An identifiable conceptual entity exposed through the API.

### What does stateless mean?

Each request should contain enough information to be processed independently without depending on hidden conversational state.

### Is REST the same as CRUD?

No.

---

## 26. Strong interview answer

> REST is an architectural style, not just a set of HTTP verbs. A REST-style API models the system around resources and representations, keeps client-server interactions stateless, uses standard HTTP semantics through a uniform interface, supports caching where appropriate, and allows layered infrastructure between client and server. CRUD maps naturally to many REST operations, but REST is broader than CRUD.

---

## Interview-Ready Summary

```text
REST
   |
   +--> Client-Server
   +--> Stateless
   +--> Cacheable
   +--> Uniform Interface
   +--> Layered System
   +--> Code on Demand (optional)

Core idea:
resources + representations + standard HTTP semantics
```

## Practice Task

Take an e-commerce backend and identify resources for:

- users
- carts
- orders
- payments
- products

Then map each resource to REST-style endpoints without using action words in URLs.
