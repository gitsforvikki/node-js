# Lesson 81 — Query Parameters in Express

## Why query parameters matter

Express makes query parameters easy to access:

```js
req.query
```

But production quality depends on validation and safe interpretation.

---

## 1. Basic example

Request:

```text
GET /users?page=2&limit=20
```

Express:

```js
app.get(
  "/users",
  (req, res) => {
    console.log(
      req.query
    );
  }
);
```

Conceptually:

```js
{
  page: "2",
  limit: "20"
}
```

---

## 2. Values need validation

Do not write:

```js
const limit =
  req.query.limit;
```

and pass it blindly into a DB query.

Convert and validate:

```js
const page =
  Math.max(
    1,
    Number(
      req.query.page ?? 1
    )
  );

const limit =
  Math.min(
    100,
    Math.max(
      1,
      Number(
        req.query.limit ?? 20
      )
    )
  );
```

---

## 3. Pagination

Request:

```text
GET /users?page=3&limit=20
```

Offset calculation:

```js
const offset =
  (page - 1) * limit;
```

Then:

```text
skip offset
take limit
```

Cursor pagination is often better for large changing datasets, covered later.

---

## 4. Filtering

```text
GET /products?category=laptop&active=true
```

Validate expected values.

Example:

```js
const active =
  req.query.active ===
  "true";
```

Do not use:

```js
Boolean(
  req.query.active
);
```

because:

```text
Boolean("false") === true
```

---

## 5. Sorting

```text
GET /products?sort=price&order=desc
```

Allowlist:

```js
const allowed =
  new Set([
    "price",
    "createdAt",
    "name",
  ]);
```

Never directly inject arbitrary query values into raw SQL or sort expressions.

---

## 6. Search

```text
GET /users?q=vikash
```

Typical flow:

```text
req.query.q
   |
   v
trim
   |
   v
validate length
   |
   v
pass to safe query layer
```

---

## 7. Repeated query values

Depending on parser/configuration, repeated query parameters may be represented differently.

Do not assume one universal type shape for complex query syntax.

For simple APIs, keep query design predictable.

---

## 8. Schema validation

Production apps often validate queries with a schema library.

Conceptually:

```text
req.query
   |
   v
schema
   |
   v
typed validated filter object
   |
   v
service/repository
```

This avoids parsing logic in controllers.

---

## 9. Query injection risk

Bad:

```js
const sort =
  req.query.sort;

db.query(
  `SELECT * FROM users ORDER BY ${sort}`
);
```

Even parameterized DB libraries may not allow identifiers to be parameterized directly.

Use allowlists.

---

## 10. Cacheability implications

GET requests with query params can still be cached.

Example:

```text
/products?page=2
```

is a distinct URL from:

```text
/products?page=3
```

Caching layers may treat them separately.

---

## 11. Secrets in query params

Avoid:

```text
/reset-password?token=super-secret
```

Sometimes systems do use temporary tokens in URLs, but understand the exposure risk:
- logs
- browser history
- analytics
- referrer leakage

For highly sensitive credentials, choose safer transport patterns.

---

## 12. Controller example

```js
export async function listUsers(
  req,
  res,
  next
) {
  try {
    const page =
      Number(
        req.query.page ?? 1
      );

    const limit =
      Number(
        req.query.limit ?? 20
      );

    const users =
      await userService.list({
        page,
        limit,
      });

    res.json(users);
  } catch (error) {
    next(error);
  }
}
```

Better still: validation middleware creates a validated query object first.

---

## 13. Common mistakes

### Mistake 1
No pagination limits.

### Mistake 2
Assuming booleans/numbers are typed.

### Mistake 3
Passing raw query values into SQL/order expressions.

### Mistake 4
Mixing query parsing with business logic everywhere.

### Mistake 5
Allowing giant search strings or unbounded filters.

---

## 14. Interview questions

### How do you access query params in Express?

Using `req.query`.

### Are they automatically typed?

No. Validate and convert them.

### Why allowlist sort fields?

To prevent invalid or unsafe query construction.

### When use query params?

For filtering, sorting, search, pagination, and optional retrieval modifiers.

---

## 15. Strong interview answer

> Express exposes query parameters through req.query, but I treat them as untrusted input. I convert types explicitly, cap pagination limits, validate allowed sort/filter fields, and ideally pass req.query through a schema validator before the controller uses it. Query parameters are best suited for optional retrieval concerns such as pagination, filtering, sorting, and search.

---

## Interview-Ready Summary

```text
req.query
   |
   +--> page
   +--> limit
   +--> filter
   +--> sort
   +--> search

Always:
convert
validate
cap
allowlist
```

## Practice Task

Implement:

```text
GET /products?page=2&limit=10&category=laptop&sort=price&order=desc
```

with validation middleware.
