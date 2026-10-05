# Lesson 93 — Resource-oriented API Design

## Why this lesson matters

Good REST APIs are designed around resources, not controller function names.

This lesson is one of the most important for API-design interviews because interviewers often ask:

> How would you design endpoints for users, orders, payments, or nested resources?

---

## 1. Think in nouns

Bad:

```text
POST /createUser
POST /getOrder
POST /deleteProduct
```

Better:

```text
POST   /users
GET    /orders/:id
DELETE /products/:id
```

The URL identifies the resource.

The method expresses the action.

---

## 2. Collection vs item resource

Collection:

```text
/users
```

Single item:

```text
/users/123
```

Typical operations:

```text
GET  /users
POST /users

GET    /users/123
PATCH  /users/123
DELETE /users/123
```

---

## 3. Resource naming

Prefer plural nouns:

```text
/users
/orders
/products
```

Consistency matters more than debating singular vs plural endlessly.

---

## 4. Avoid verbs in resource paths

Avoid:

```text
/users/create
/orders/delete
/products/update
```

The HTTP method already conveys the operation.

---

## 5. Nested resources

Example:

```text
/users/123/orders
```

This communicates:

> orders belonging to user 123.

Good when hierarchy adds meaning.

---

## 6. Avoid excessive nesting

Bad:

```text
/companies/1/departments/2/teams/3/users/4/orders/5
```

Too deep.

Often once a resource has its own ID:

```text
/orders/5
```

is enough.

---

## 7. Parent-child ownership

Nested endpoint:

```text
GET /users/123/orders/456
```

The query should usually enforce:

```text
order.id = 456
AND
order.userId = 123
```

Do not trust path shape alone.

---

## 8. Actions that do not fit CRUD

Suppose:

```text
cancel order
```

Possible design approaches:

### State update

```http
PATCH /orders/123

{
  "status": "cancelled"
}
```

### Action as subresource

```text
POST /orders/123/cancellations
```

Which is better depends on domain semantics.

---

## 9. Model events/resources, not functions

Instead of:

```text
POST /orders/123/cancel
```

you can model cancellation as a resource:

```text
POST /orders/123/cancellations
```

This works well when cancellation itself has:
- timestamp
- reason
- actor
- status

---

## 10. Search endpoint design

Simple search:

```text
GET /products?q=laptop
```

Complex search may justify:

```text
POST /product-searches
```

if the filter object is too complex or too large for a URL.

Resource-oriented design still applies.

---

## 11. Filters belong in query params

Example:

```text
GET /orders?status=paid&userId=123
```

Avoid:

```text
/getPaidOrdersForUser/123
```

---

## 12. Pagination

Collection endpoint:

```text
GET /orders?page=2&limit=20
```

or cursor-based:

```text
GET /orders?cursor=abc&limit=20
```

Pagination is part of collection retrieval, not a separate resource action.

---

## 13. Sorting

```text
GET /products?sort=price&order=asc
```

Allowlist sort fields.

Do not pass arbitrary client input directly into DB ordering.

---

## 14. Field selection

Some APIs support:

```text
GET /users/123?fields=id,name,email
```

Useful for bandwidth control, though it increases API complexity.

---

## 15. Relationships

Example:

```text
GET /orders/123/items
```

Good when order items are a meaningful subresource.

---

## 16. Stable resource IDs

URLs should use identifiers that remain stable.

Bad:

```text
/users/current-email@example.com
```

if email changes.

Better:

```text
/users/123
```

---

## 17. Idempotency and resource design

Creation with client-generated ID can sometimes be modeled as:

```http
PUT /users/123
```

because the client knows the final resource URI.

Creation with server-generated ID commonly uses:

```http
POST /users
```

---

## 18. Response shape

Collection:

```json
{
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

Single resource:

```json
{
  "data": {
    "id": 123,
    "name": "Vikash"
  }
}
```

Consistency matters.

---

## 19. Do not expose DB schema directly

Database:

```text
usr_tbl
usr_nm
pwd_hash
```

API:

```json
{
  "id": 123,
  "name": "Vikash"
}
```

API contracts should reflect domain language, not storage internals.

---

## 20. Resource design for payments

Example:

```text
POST /payments
GET  /payments/:id
```

For capture/refund flows, think carefully about whether they are:
- state transitions
- subresources
- explicit domain operations

Payment APIs often need idempotency and auditability more than stylistic purity.

---

## 21. Common mistakes

### Mistake 1
Action-heavy URLs.

### Mistake 2
Over-nesting.

### Mistake 3
Exposing DB naming/storage details.

### Mistake 4
Treating query filters as path hierarchy.

### Mistake 5
Ignoring ownership relationships.

---

## 22. Interview questions

### Why use nouns in REST URLs?

Because URLs identify resources, while HTTP methods communicate operations.

### When should resources be nested?

When the parent-child relationship is meaningful for addressing or authorization.

### How deep should nesting go?

Prefer shallow nesting; once a resource has a globally unique identity, direct access is often cleaner.

### How would you model order cancellation?

Either as a status transition or as a cancellation subresource depending on domain requirements.

---

## 23. Strong interview answer

> Resource-oriented API design models stable domain nouns such as users, orders, and payments rather than controller actions. I distinguish collection and item resources, keep nesting shallow, use query parameters for filtering/sorting/pagination, and enforce parent-child ownership when nested routes are used. For operations that do not fit CRUD cleanly, I model state transitions or subresources based on the domain instead of blindly adding verb endpoints.

---

## Interview-Ready Summary

```text
Good resource API
   |
   +--> nouns
   +--> collections
   +--> item resources
   +--> shallow nesting
   +--> query-based filtering
   +--> stable identifiers
   +--> domain-oriented naming
```

## Practice Task

Design REST endpoints for:

- users
- friendships
- products
- carts
- orders
- payments
- refunds

Explain why each endpoint is a resource or subresource.
