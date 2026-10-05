# Lesson 98 — Filtering

## Why filtering matters

Filtering lets clients retrieve only the resources they need.

Example:

```text
GET /orders?status=paid&userId=123
```

A good filtering design should be:

- predictable
- validated
- secure
- index-friendly
- easy to extend

---

## 1. Basic filtering

Request:

```text
GET /products?category=laptop&brand=apple
```

Conceptual filter:

```js
{
  category: "laptop",
  brand: "apple",
}
```

---

## 2. Never pass req.query directly to the database

Bad:

```js
repository.find(
  req.query
);
```

Why?

The client may send:
- unsupported fields
- internal fields
- dangerous operators
- unexpected data types

Always construct an allowlisted filter object.

---

## 3. Allowlist fields

Example:

```js
const allowedFilters =
  new Set([
    "status",
    "category",
    "brand",
  ]);
```

Only accepted fields should reach the repository layer.

---

## 4. Type validation

Query parameters commonly arrive as strings.

Example:

```text
?minPrice=1000&active=false
```

Validate/convert explicitly.

Be careful:

```js
Boolean("false")
```

returns:

```text
true
```

Use explicit boolean parsing.

---

## 5. Equality filters

Example:

```text
GET /orders?status=paid
```

Database concept:

```sql
WHERE status = 'paid'
```

Simple and index-friendly when indexed appropriately.

---

## 6. Range filtering

Example:

```text
GET /products?minPrice=1000&maxPrice=5000
```

Database:

```sql
WHERE price >= 1000
AND price <= 5000
```

---

## 7. Date ranges

Example:

```text
GET /orders?from=2026-01-01&to=2026-01-31
```

Be explicit about:
- timezone
- inclusive/exclusive boundaries
- date format

For timestamps, half-open intervals are often useful:

```text
createdAt >= from
createdAt < to
```

This reduces end-of-day ambiguity.

---

## 8. Multi-value filters

Example:

```text
GET /products?category=laptop&category=mobile
```

or:

```text
GET /products?category=laptop,mobile
```

Pick one convention and document it.

Database concept:

```sql
WHERE category IN (...)
```

---

## 9. Filter operators

More advanced APIs may support:

```text
price[gte]=1000
price[lte]=5000
```

or:

```text
filter[price][gte]=1000
```

This is expressive but increases parsing and security complexity.

Use only if your API genuinely needs it.

---

## 10. Avoid exposing DB operators

Never accept arbitrary MongoDB-style operators directly from clients.

Bad:

```json
{
  "$where": "...",
  "$ne": null
}
```

Build safe application-level filter semantics and map them internally.

---

## 11. Filtering by ownership

Authenticated APIs often need an implicit filter.

Example:

```text
GET /orders
```

Authenticated user ID:

```text
123
```

Repository query:

```text
WHERE user_id = 123
```

Do not rely on clients to send:

```text
?userId=123
```

when ownership should come from auth context.

---

## 12. Admin filters vs user filters

Admin:

```text
GET /admin/orders?userId=123
```

may allow broader filters.

User endpoint:

```text
GET /orders
```

should usually scope automatically to the authenticated user.

---

## 13. Filter composition

Example:

```text
status=paid
category=subscription
from=...
to=...
```

These often combine with AND logic.

Document if OR behavior is supported.

Ambiguous filtering semantics create client bugs.

---

## 14. Filtering and indexes

Suppose common query:

```text
WHERE user_id = ?
AND status = ?
ORDER BY created_at DESC
```

A composite index may be useful:

```text
(user_id, status, created_at)
```

Do not add indexes blindly; design them around actual query patterns.

---

## 15. High-cardinality vs low-cardinality fields

Filtering on:
- user_id
- email
- order_id

is usually selective.

Filtering only on:
- active=true
- country
- status

may match many rows.

Index usefulness depends on data distribution and query patterns.

---

## 16. Avoid unbounded filter complexity

Allowing clients to create arbitrary nested logical expressions can lead to:

- expensive DB queries
- difficult caching
- hard-to-predict performance

Public APIs should usually expose controlled filter capabilities.

---

## 17. Filtering and pagination

Filtering must happen **before** pagination.

Correct:

```text
filter
  |
  v
sort
  |
  v
paginate
```

Not:

```text
paginate all rows
  |
  v
filter in Node.js
```

The database should do filtering whenever possible.

---

## 18. Filtering and total count

If you return total:

```text
total
```

it must count the **filtered** set, not the entire table.

That extra count may be expensive.

---

## 19. Response example

Request:

```text
GET /products?category=laptop&minPrice=50000
```

Response:

```json
{
  "data": [],
  "meta": {
    "filters": {
      "category": "laptop",
      "minPrice": 50000
    }
  }
}
```

Returning applied filters is optional but can be useful.

---

## 20. Common mistakes

### Mistake 1
Passing req.query directly into DB.

### Mistake 2
No allowlist.

### Mistake 3
No type conversion.

### Mistake 4
Allowing arbitrary database operators.

### Mistake 5
Filtering in application memory after fetching huge datasets.

### Mistake 6
Trusting client ownership filters.

---

## 21. Interview questions

### How should filters be designed?

Expose a controlled set of documented query parameters, validate them, and map them to safe DB predicates.

### Why not pass req.query directly to MongoDB/SQL layer?

It can expose unsupported or dangerous operators/fields and create injection/security issues.

### Where should filtering happen?

Usually in the database, before sorting and pagination.

### How does indexing relate to filtering?

Frequently used filter/sort combinations should guide index design.

---

## 22. Strong interview answer

> I expose only an allowlisted set of filter parameters, validate and normalize their types, and translate them into safe repository predicates. I do not pass req.query directly to the database or expose raw database operators. Filtering should happen in the database before pagination, and I design indexes around the filter and sort combinations that are actually common in production.

---

## Interview-Ready Summary

```text
Filtering
   |
   +--> allowlist
   +--> validate
   +--> normalize types
   +--> map to DB query
   +--> ownership scope
   +--> indexes

Order:
filter -> sort -> paginate
```

## Practice Task

Design filters for:

```text
GET /orders
```

Support:
- status
- paymentStatus
- minAmount
- maxAmount
- from
- to

Then explain which filters should come from auth context vs query params.
