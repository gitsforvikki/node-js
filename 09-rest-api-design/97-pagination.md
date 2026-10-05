# Lesson 97 — Pagination

## Why pagination matters

Returning an entire collection is one of the fastest ways to make an API slow and unstable.

Imagine:

```text
GET /users
```

and the database contains:

```text
10,000,000 users
```

Returning everything would cause:

- huge database scans
- large memory usage
- slow JSON serialization
- large network payloads
- high latency
- possible process instability

Pagination solves this by returning data in smaller chunks.

---

## 1. What is pagination?

Pagination divides a large result set into smaller pages.

Example:

```text
GET /users?page=2&limit=20
```

Conceptually:

```text
records 1-20   -> page 1
records 21-40  -> page 2
records 41-60  -> page 3
```

---

# Offset Pagination

## 2. Offset-based pagination

Common parameters:

```text
page
limit
```

or:

```text
offset
limit
```

Example:

```text
GET /users?page=3&limit=20
```

Offset:

```text
(page - 1) * limit

(3 - 1) * 20
= 40
```

Database concept:

```sql
SELECT *
FROM users
ORDER BY created_at DESC
LIMIT 20
OFFSET 40;
```

---

## 3. Advantages of offset pagination

- easy to understand
- easy to implement
- supports page numbers
- useful for admin tables
- easy "jump to page 20"

---

## 4. Problems with large offsets

Suppose:

```sql
OFFSET 1000000
```

Many databases still need to walk past a large number of rows before returning the requested result.

That can become expensive.

Mental model:

```text
scan many rows
     |
     v
discard first 1,000,000
     |
     v
return next 20
```

---

## 5. Data-shift problem

Suppose page 1 returns:

```text
A
B
C
D
E
```

Before page 2 is requested, a new row X is inserted at the top.

Now:

```text
X
A
B
C
D
E
...
```

Offset pagination may cause:
- duplicates
- skipped records

This matters in frequently changing feeds.

---

# Cursor Pagination

## 6. What is cursor pagination?

Instead of asking for page 5, the client says:

> Give me the next records after this known position.

Example:

```text
GET /users?after=cursor123&limit=20
```

---

## 7. Cursor mental model

```text
Page 1
A
B
C
D
E
   |
   v
cursor points after E

Next request:
after=E
   |
   v
F
G
H
I
J
```

---

## 8. Keyset pagination

A common cursor implementation uses indexed ordering fields.

Example:

```sql
SELECT *
FROM users
WHERE created_at < $1
ORDER BY created_at DESC
LIMIT 20;
```

This avoids large OFFSET scans.

---

## 9. Tie-breaking is critical

If multiple rows have identical:

```text
created_at
```

then sorting only by `created_at` may be unstable.

Better:

```sql
ORDER BY created_at DESC, id DESC
```

Cursor should encode both:

```text
created_at
id
```

This creates deterministic ordering.

---

## 10. Stable sort is mandatory

Pagination without deterministic ordering is unsafe.

Bad:

```sql
SELECT *
FROM users
LIMIT 20;
```

Database is not required to return rows in a stable order.

Always define ordering.

---

## 11. Cursor encoding

Avoid exposing implementation details directly if unnecessary.

Instead of:

```text
?createdAt=2026-01-01&id=999
```

you can return an opaque cursor.

Example conceptual payload:

```json
{
  "createdAt": "2026-01-01T10:00:00Z",
  "id": 999
}
```

Encode:

```text
base64url(...)
```

Important:

Base64 is encoding, not security.

If tamper-resistance matters, sign the cursor or validate it strictly.

---

## 12. Cursor response example

```json
{
  "data": [],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIzfQ",
    "hasMore": true
  }
}
```

---

## 13. Offset response example

```json
{
  "data": [],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 421,
    "totalPages": 22
  }
}
```

---

## 14. Total count trade-off

Clients often want:

```text
total = 10,000,000
```

But:

```sql
COUNT(*)
```

can itself be expensive on large filtered datasets.

Do not assume exact total is free.

Alternatives:
- omit total
- cache count
- approximate count
- return `hasMore`

---

## 15. Page size limits

Never trust:

```text
?limit=1000000
```

Example policy:

```text
default limit = 20
max limit = 100
```

This protects:
- DB
- memory
- bandwidth

---

## 16. Validation

Example:

```js
const page =
  Math.max(
    1,
    Number(req.query.page ?? 1)
  );

const limit =
  Math.min(
    100,
    Math.max(
      1,
      Number(req.query.limit ?? 20)
    )
  );
```

A schema validator is better for production.

---

## 17. Pagination + filtering

Example:

```text
GET /orders?status=paid&limit=20&after=abc
```

The cursor must correspond to the exact filter/sort context.

Otherwise:

```text
cursor created for:
status=paid

reused with:
status=pending
```

can produce incorrect results.

A robust cursor can encode:
- sort values
- optional filter fingerprint

---

## 18. Pagination + sorting

Cursor logic must match sort direction.

For descending:

```sql
WHERE created_at < cursor_created_at
ORDER BY created_at DESC
```

For ascending:

```sql
WHERE created_at > cursor_created_at
ORDER BY created_at ASC
```

---

## 19. Bidirectional pagination

Some APIs support:

```text
before
after
```

Useful for:
- chat history
- timelines
- feeds

This requires careful reverse ordering.

---

## 20. Chat example

Suppose messages are ordered newest first.

Request:

```text
GET /messages?before=<messageCursor>&limit=50
```

This is usually better than page numbers because new messages are constantly added.

---

## 21. Pagination and indexes

A cursor query is only fast if the database has a useful index.

Example:

```text
ORDER BY created_at DESC, id DESC
```

Ideal index concept:

```text
(created_at, id)
```

Pagination design and indexing should be considered together.

---

## 22. Offset vs cursor comparison

| Feature | Offset | Cursor |
|---|---|---|
| Easy to implement | Yes | More complex |
| Jump to page N | Yes | Usually no |
| Large dataset efficiency | Weaker | Better |
| Stable under inserts | Weaker | Better |
| Infinite feed | Okay | Excellent |
| Admin table | Excellent | Sometimes unnecessary |

---

## 23. When to use offset

Use offset/page pagination when:

- dataset is moderate
- users need page numbers
- admin dashboard/table
- data is relatively stable
- implementation simplicity matters

---

## 24. When to use cursor

Use cursor pagination when:

- dataset is large
- feed changes frequently
- infinite scrolling
- chat/history
- performance at deep pages matters

---

## 25. Common mistakes

### Mistake 1
No ORDER BY.

### Mistake 2
No maximum page size.

### Mistake 3
Large OFFSET on huge datasets.

### Mistake 4
Cursor based on non-unique sort field only.

### Mistake 5
Cursor reused with different filters/sorting.

### Mistake 6
Assuming total count is free.

---

## 26. Interview questions

### Offset vs cursor pagination?

Offset uses row position and supports page numbers but becomes less efficient and less stable for deep, changing datasets. Cursor pagination continues after a known ordered record and is generally better for large feeds.

### Why must pagination have deterministic ordering?

Without stable ordering, records can move unpredictably between requests, causing duplicates or missing items.

### Why add ID as a tie-breaker?

Multiple rows can share the same primary sort value, so a unique secondary sort key makes ordering deterministic.

### Is cursor pagination automatically fast?

No. It still needs proper indexing.

---

## 27. Strong interview answer

> For small or admin-style datasets, offset pagination is simple and supports page numbers. For large or frequently changing feeds, I prefer cursor/keyset pagination because it avoids deep OFFSET scans and is more stable under inserts. I always use deterministic ordering, usually with a unique tie-breaker like id, enforce a maximum page size, and align the database index with the cursor sort fields.

---

## Interview-Ready Summary

```text
Offset Pagination
  -> page + limit
  -> simple
  -> deep pages expensive

Cursor Pagination
  -> after/before cursor
  -> scalable
  -> stable for feeds

Always:
limit max
stable ORDER BY
unique tie-breaker
proper index
```

## Practice Task

Design pagination for:

1. admin users table
2. Instagram-style feed
3. chat messages
4. order history

Choose offset or cursor and explain why.
