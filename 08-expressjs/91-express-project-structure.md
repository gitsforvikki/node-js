# Lesson 91 — Structuring Large Express Applications

## Why this lesson matters

A large Express application becomes difficult to maintain if everything lives in:

```text
app.js
```

The framework is intentionally unopinionated, so architecture is your responsibility.

This lesson shows a practical structure suitable for production and interviews.

---

## 1. Core principle

Separate concerns.

A strong architecture often separates:

```text
HTTP
Business Logic
Data Access
Infrastructure
Configuration
Cross-cutting Concerns
```

---

## 2. Example project structure

```text
src/
├── app.js
├── server.js
│
├── config/
│   ├── env.js
│   ├── database.js
│   └── logger.js
│
├── routes/
│   ├── index.js
│   ├── user.routes.js
│   └── order.routes.js
│
├── controllers/
│   ├── user.controller.js
│   └── order.controller.js
│
├── services/
│   ├── user.service.js
│   └── order.service.js
│
├── repositories/
│   ├── user.repository.js
│   └── order.repository.js
│
├── middlewares/
│   ├── auth.middleware.js
│   ├── validate.middleware.js
│   ├── error.middleware.js
│   └── request-id.middleware.js
│
├── validators/
│   ├── user.validator.js
│   └── order.validator.js
│
├── errors/
│   └── app-error.js
│
├── utils/
│   └── helpers.js
│
└── integrations/
    ├── email/
    └── payment/
```

---

## 3. app.js responsibility

`app.js` should configure the Express application.

Example responsibilities:

- built-in middleware
- security middleware
- request logging
- routers
- 404 handler
- error handler

It should not usually:
- connect DB
- listen on port
- contain business logic

---

## 4. server.js responsibility

`server.js` handles runtime startup.

Typical flow:

```text
load config
   |
   v
connect database
   |
   v
connect Redis
   |
   v
start HTTP server
   |
   v
handle shutdown signals
```

This separation improves testing.

---

## 5. Routes responsibility

Routes declare:

```text
method
path
middleware
controller
```

Example:

```js
router.post(
  "/",
  authenticate,
  validateCreateOrder,
  createOrder
);
```

Do not place business logic here.

---

## 6. Controllers responsibility

Controllers translate HTTP into application calls.

They usually:

- read params/query/body
- call service
- map result to HTTP response
- pass errors onward

Example:

```js
export async function createOrder(
  req,
  res
) {
  const order =
    await orderService.create({
      userId:
        req.user.id,
      input:
        req.validatedBody,
    });

  res
    .status(201)
    .json(order);
}
```

---

## 7. Services responsibility

Services contain business logic.

Example:

```text
create order
   |
   +--> validate business rules
   +--> reserve inventory
   +--> calculate totals
   +--> create payment intent
   +--> persist order
```

Services should not depend heavily on Express `req` or `res`.

---

## 8. Repositories responsibility

Repositories encapsulate data access.

Example:

```js
userRepository.findById(
  id
);

userRepository.create(
  data
);
```

Benefits:
- DB logic centralized
- easier testing
- easier migrations
- services stay domain-focused

---

## 9. Validators

Validation belongs before business logic.

Example:

```text
req.body
   |
   v
schema validator
   |
   v
validated DTO
   |
   v
controller/service
```

This prevents unsafe raw input from entering the service layer.

---

## 10. Middleware

Middleware handles request pipeline concerns:

- authentication
- authorization
- request IDs
- rate limiting
- validation
- logging

Avoid putting transactional business workflows here.

---

## 11. Integrations

External systems deserve their own boundary.

Examples:

```text
payment/
email/
sms/
storage/
analytics/
```

Instead of:

```js
fetch("payment-provider")
```

scattered throughout services.

---

## 12. Config module

Avoid:

```js
process.env.JWT_SECRET
```

everywhere.

Prefer centralized validated configuration.

Conceptually:

```js
export const config = {
  port,
  databaseUrl,
  jwtSecret,
};
```

Validate required environment variables at startup.

---

## 13. Feature-based structure

Another valid pattern groups by feature.

```text
src/
├── users/
│   ├── user.routes.js
│   ├── user.controller.js
│   ├── user.service.js
│   ├── user.repository.js
│   └── user.validator.js
│
├── orders/
│   ├── order.routes.js
│   ├── order.controller.js
│   ├── order.service.js
│   └── order.repository.js
```

This often scales better in very large applications.

---

## 14. Layer-based vs feature-based

Layer-based:

```text
controllers/
services/
repositories/
```

Feature-based:

```text
users/
orders/
payments/
```

Neither is universally correct.

Choose based on:
- team size
- app size
- domain boundaries
- ownership model

---

## 15. Recommended hybrid

A practical hybrid:

```text
modules/
  users/
    routes
    controller
    service
    repository
    validator

shared/
  middleware
  config
  errors
  logger
```

This keeps features cohesive while sharing infrastructure.

---

## 16. Dependency direction

A healthy dependency flow:

```text
Routes
   |
   v
Controllers
   |
   v
Services
   |
   v
Repositories / Integrations
```

Avoid reverse dependencies such as repositories importing controllers.

---

## 17. Keep Express at the edge

A strong design keeps Express-specific objects near the HTTP layer.

Bad:

```js
function createOrder(
  req,
  res
) {
  // service logic
}
```

inside a service.

Better:

```js
orderService.create({
  userId,
  items,
});
```

This makes services reusable and testable.

---

## 18. Transactions

Transaction boundaries usually belong around business workflows, often in the service layer.

Example:

```text
Service
   |
   v
begin transaction
   |
   +--> repository A
   +--> repository B
   |
   v
commit / rollback
```

Do not scatter transaction logic across controllers.

---

## 19. Error architecture

```text
Repository error
    |
    v
Service translation
    |
    v
Controller propagation
    |
    v
Error middleware
```

This prevents raw infrastructure errors from leaking to clients.

---

## 20. Testing architecture

Good separation enables:

- unit-test services
- integration-test repositories
- API-test routers/controllers
- mock external integrations

If every layer directly depends on req/res/DB, testing becomes harder.

---

## 21. Logging and observability

Shared infrastructure should handle:

- structured logger
- request ID
- timing
- metrics
- tracing hooks

Do not scatter `console.log` throughout production code.

---

## 22. Example request flow

```text
POST /api/v1/orders
        |
        v
Router
        |
        v
Auth Middleware
        |
        v
Validation
        |
        v
Controller
        |
        v
Order Service
        |
        +--> Order Repository
        +--> Payment Integration
        +--> Inventory Repository
        |
        v
Controller Response
        |
        v
Error Middleware if needed
```

---

## 23. Anti-pattern: god service

Bad:

```text
app.js
  routes
  DB calls
  payment
  email
  validation
  auth
  logging
```

This becomes impossible to scale safely.

---

## 24. Anti-pattern: too many layers

Do not create abstractions just for appearance.

For a tiny app:

```text
route
controller
service
repository
factory
adapter
manager
facade
helper
```

may be unnecessary.

Architecture should reduce complexity, not create ceremony.

---

## 25. Production startup flow

A solid startup sequence:

```text
load + validate env
      |
      v
initialize logger
      |
      v
connect database
      |
      v
connect Redis / dependencies
      |
      v
create server
      |
      v
listen
      |
      v
ready
```

Shutdown:

```text
SIGTERM
   |
   v
stop new requests
   |
   v
finish active work
   |
   v
close DB/Redis
   |
   v
exit
```

---

## 26. Common mistakes

### Mistake 1
Business logic in controllers.

### Mistake 2
Database queries in route files.

### Mistake 3
Services depending on Express req/res.

### Mistake 4
No configuration validation.

### Mistake 5
One huge global utils file.

### Mistake 6
Excessive architecture for a tiny application.

---

## 27. Interview questions

### How do you structure a large Express app?

Separate HTTP routing/controllers, business services, data repositories, middleware, validation, configuration, and external integrations.

### Why keep Express out of services?

It improves testability and keeps business logic independent of the transport framework.

### Layer-based vs feature-based structure?

Both are valid; feature-based or hybrid structures often scale better as domains grow.

### Where should transactions live?

Usually around business workflows in the service/application layer.

---

## 28. Strong interview answer

> For a large Express application, I keep the HTTP layer thin. Routes define method, path, middleware, and controller. Controllers translate HTTP input/output, services contain business logic, repositories handle data access, and integrations wrap external systems. I keep configuration and middleware shared, validate environment variables at startup, and usually prefer a feature-based or hybrid structure as the codebase grows. The goal is clear dependency direction and testable business logic, not adding layers just for ceremony.

---

## Interview-Ready Summary

```text
Route
  -> endpoint definition

Controller
  -> HTTP translation

Service
  -> business logic

Repository
  -> data access

Integration
  -> external systems

Middleware
  -> request concerns

Config
  -> validated environment
```

## Section 8 Final Mental Model

```text
Client
   |
   v
Express App
   |
   +--> Global Middleware
   |
   +--> Version Router
   |
   +--> Feature Router
   |
   +--> Auth / Validation
   |
   +--> Controller
   |
   +--> Service
   |
   +--> Repository / Integration
   |
   v
Response

Any failure
   |
   v
Central Error Middleware
```

Section 8 is now complete conceptually from basic Express usage through production application architecture.

## Practice Task

Design a production Express project for an e-commerce backend containing:

- auth
- users
- products
- orders
- payments
- notifications

Show:
- folder structure
- request flow
- where validation lives
- where DB queries live
- where payment integration lives
- where transactions belong
- where error handling lives
