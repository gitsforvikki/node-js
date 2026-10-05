# Lesson 99 — Sorting

## Why sorting matters

Sorting is required for:

- predictable result order
- stable pagination
- user-controlled tables
- feeds
- reports

A collection endpoint without deterministic sorting can produce inconsistent results.

---

## 1. Basic sorting

Example:

```text
GET /products?sort=price&order=asc
```

Database concept:

```sql
ORDER BY price ASC
```

---

## 2. Default sort

Every paginated collection should usually have a default deterministic sort.

Example:

```text
createdAt DESC
id DESC
```

This gives stable ordering.

---

## 3. Why deterministic ordering matters

Suppose you sort only by:

```text
createdAt DESC
```

and 20 rows have the same timestamp.

Their relative order may change.

Add a tie-breaker:

```text
createdAt DESC, id DESC
```

This is especially important for cursor pagination.

---

## 4. Allowlist sort fields

Bad:

```js
const sort =
  req.query.sort;

sql =
  `ORDER BY ${sort}`;
```

Client could inject or request internal/expensive fields.

Use an allowlist:

```js
const sortMap = {
  price: "price",
  createdAt: "created_at",
  name: "name",
};
```

---

## 5. Validate direction

Allow:

```text
asc
desc
```

Normalize case if desired.

Reject everything else.

---

## 6. Multi-column sort

Possible design:

```text
GET /users?sort=-createdAt,name
```

Convention:

```text
-createdAt -> DESC
name       -> ASC
```

This is expressive but requires strict parsing.

---

## 7. Default tie-breaker

Even when user asks:

```text
sort=price
```

consider internally adding:

```text
id
```

to ensure deterministic results.

Example:

```sql
ORDER BY price ASC, id ASC
```

---

## 8. Sorting NULL values

Different DBs may place NULLs differently.

You may need explicit behavior:

```sql
ORDER BY published_at DESC NULLS LAST
```

Be deliberate if clients depend on ordering.

---

## 9. Case-insensitive sorting

Sorting names can be tricky.

```text
Apple
banana
Carrot
```

Collation affects order.

Database collation and locale rules may matter.

Do not assume ASCII order equals user-friendly order.

---

## 10. Locale-aware sorting

For user-facing text, language/collation requirements can matter.

Example:
- English
- Hindi
- German
- Turkish

For many APIs, DB default collation is acceptable, but understand that text sorting is not universally trivial.

---

## 11. Sorting computed fields

Example:

```text
sort=totalSpent
```

If `totalSpent` requires aggregation, this can be expensive.

Do not expose arbitrary computed sorting without understanding cost.

---

## 12. Sorting and indexes

Query:

```sql
WHERE user_id = ?
ORDER BY created_at DESC
```

Index concept:

```text
(user_id, created_at)
```

can improve performance.

---

## 13. Composite index order matters

Suppose index:

```text
(status, created_at)
```

This may help:

```text
WHERE status = 'paid'
ORDER BY created_at DESC
```

Index design should match actual query patterns.

---

## 14. Sorting and cursor pagination

Cursor condition must match ordering.

For:

```text
ORDER BY created_at DESC, id DESC
```

cursor predicate concept:

```text
created_at < cursorCreatedAt

OR

created_at = cursorCreatedAt
AND id < cursorId
```

This is the essence of compound keyset pagination.

---

## 15. Sorting and filtering order

Database logical flow:

```text
filter matching rows
      |
      v
sort
      |
      v
limit/paginate
```

Do not fetch arbitrary rows first and sort them in Node.js unless the dataset is intentionally small.

---

## 16. User-friendly API design

Option A:

```text
?sort=price&order=asc
```

Option B:

```text
?sort=price:asc
```

Option C:

```text
?sort=price
```

with predefined order.

Pick one convention and document it.

---

## 17. Security

Raw SQL identifiers cannot always be parameterized like values.

So:

```text
ORDER BY ?
```

often does not work the same way as:

```text
WHERE id = ?
```

This makes allowlisting sort fields especially important.

---

## 18. Common mistakes

### Mistake 1
Raw client sort string inserted into SQL.

### Mistake 2
No stable tie-breaker.

### Mistake 3
Sorting in memory after huge DB fetch.

### Mistake 4
Allowing expensive computed sorts freely.

### Mistake 5
Ignoring NULL/collation behavior.

---

## 19. Interview questions

### Why allowlist sort fields?

Because dynamic identifiers are harder to parameterize and arbitrary fields can create security/performance issues.

### Why add a tie-breaker?

To make ordering deterministic when primary sort values are equal.

### How does sorting affect cursor pagination?

The cursor must encode and compare the same ordered fields.

### Why should sorting happen in DB?

Databases can use indexes and avoid transferring unnecessary rows.

---

## 20. Strong interview answer

> I expose a small allowlisted set of sort fields and directions, apply a deterministic default order, and add a unique tie-breaker such as id. Sorting happens in the database before pagination so indexes can be used. For cursor pagination, the cursor must follow the exact same composite ordering; otherwise records can be skipped or duplicated.

---

## Interview-Ready Summary

```text
Sorting
   |
   +--> allowlist fields
   +--> validate direction
   +--> stable default
   +--> unique tie-breaker
   +--> DB-side execution
   +--> index awareness

Cursor pagination requires
same sort definition.
```

## Practice Task

Design sorting for:

```text
GET /products
```

Support:
- price
- rating
- createdAt
- name

Define:
- default sort
- tie-breaker
- allowed directions
- recommended indexes
