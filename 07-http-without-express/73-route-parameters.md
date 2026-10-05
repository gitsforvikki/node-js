# Lesson 73 — Route Parameters

## Why route parameters matter

Route parameters identify a specific resource within a URL path.

Examples:

```text
/users/123
/orders/abc123
/products/42/reviews/9
```

Frameworks make them look simple:

```js
req.params.id
```

But raw Node.js requires you to understand how the path is actually matched.

---

## 1. What is a route parameter?

Given:

```text
/users/123
```

The conceptual route pattern is:

```text
/users/:id
```

where:

```text
id = 123
```

---

## 2. Route params identify resources

Example:

```text
GET /users/123
```

means:

> Give me user 123.

This differs from:

```text
GET /users?page=2
```

which means:

> Give me the users collection, but return page 2.

---

## 3. Extracting manually

First parse pathname:

```js
const url =
  new URL(
    req.url,
    `http://${req.headers.host}`
  );

const pathname =
  url.pathname;
```

For:

```text
/users/123
```

you can split:

```js
const segments =
  pathname
    .split("/")
    .filter(Boolean);
```

Result:

```js
[
  "users",
  "123"
]
```

---

## 4. Basic match

```js
if (
  req.method === "GET" &&
  segments[0] === "users" &&
  segments.length === 2
) {
  const userId =
    segments[1];
}
```

---

## 5. Decode path segments

URLs may contain encoded values.

Use:

```js
decodeURIComponent(
  segments[1]
);
```

Do not assume raw encoded path values are already application-ready.

---

## 6. Validate route parameters

Example:

```text
/users/abc
```

If user IDs must be numeric:

```js
const id =
  Number(
    segments[1]
  );

if (
  !Number.isInteger(id) ||
  id <= 0
) {
  // 400
}
```

---

## 7. UUID validation

If identifiers are UUIDs:

```text
/users/550e8400-e29b-41d4-a716-446655440000
```

validate format before database access.

Do not simply trust the string.

---

## 8. Nested resources

Example:

```text
/users/123/orders/456
```

Conceptual params:

```text
userId = 123
orderId = 456
```

Possible meaning:

> Fetch order 456 belonging to user 123.

---

## 9. Authorization matters

This is a very important production issue.

Suppose:

```text
GET /users/123/orders/456
```

Do not only query:

```text
order id = 456
```

You may need to enforce:

```text
order.id = 456
AND
order.userId = 123
```

Otherwise users may access resources they do not own.

This is related to broken object-level authorization.

---

## 10. Resource hierarchy

Good:

```text
/users/123/orders
/orders/456
```

Avoid unnecessarily deep routes like:

```text
/companies/1/departments/2/teams/3/users/4/orders/5
```

Deep nesting makes APIs harder to maintain.

---

## 11. Route parameter vs body

Use the route parameter for resource identity:

```text
PATCH /users/123
```

Body:

```json
{
  "name": "Vikash"
}
```

Do not require duplicated ID unless there is a clear reason.

---

## 12. Manual pattern matching

A reusable matcher concept:

```text
Pattern:
/users/:id

Actual:
/users/123

Compare:
users == users
:id captures 123
```

This is the basic idea behind framework routers.

---

## 13. Regex approach

Conceptually:

```js
const match =
  pathname.match(
    /^\/users\/([^/]+)$/
  );

if (match) {
  const id =
    decodeURIComponent(
      match[1]
    );
}
```

For small exercises this works.

For production-scale routing, frameworks provide safer abstractions.

---

## 14. Common mistakes

### Mistake 1
Not validating IDs.

### Mistake 2
Ignoring URL decoding.

### Mistake 3
Fetching nested resources without ownership constraints.

### Mistake 4
Over-nesting routes.

### Mistake 5
Confusing route params and query params.

---

## 15. Interview questions

### What is a route parameter?

A dynamic path segment used to identify a particular resource.

### How do route params differ from query params?

Route params identify resource identity; query params usually modify filtering, sorting, pagination, or search.

### Why validate route params before DB queries?

To reject malformed input early and reduce unnecessary or unsafe database access.

### What security issue can occur with object IDs?

Broken object-level authorization if users can access IDs they do not own.

---

## 16. Strong interview answer

> Route parameters are dynamic URL path segments used to identify resources, such as the 123 in /users/123. In raw Node.js, I parse the pathname and match its segments manually. I validate and decode route parameters before using them, and for nested or user-owned resources I enforce ownership in the database query rather than trusting the path alone.

---

## Interview-Ready Summary

```text
/users/:id
       |
       v
resource identifier

Important:
decode
validate
authorize
avoid excessive nesting
```

## Practice Task

Implement manually:

```text
GET /users/:userId
GET /users/:userId/orders/:orderId
```

Validate both IDs and ensure an order belongs to the specified user.
