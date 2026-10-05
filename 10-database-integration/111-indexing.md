# Lesson 111 — Indexing

## Why indexing is one of the most important database topics

Most database performance problems eventually involve:

- missing indexes
- wrong indexes
- too many indexes
- queries that cannot use indexes

An index can turn:

```text
scan 10 million documents
```

into:

```text
find matching keys quickly
```

But indexes are not free.

---

## 1. What is an index?

An index is an auxiliary data structure that helps the database locate documents efficiently.

Mental model:

Without index:

```text
Collection
A
B
C
D
E
...
scan until match
```

With index:

```text
Index
email -> document location
```

---

## 2. Collection scan

Example:

```js
User.findOne({
  email:
    "vikash@example.com",
});
```

Without email index, MongoDB may scan many documents.

This is:

```text
COLLSCAN
```

---

## 3. Indexed query

With:

```js
userSchema.index({
  email: 1,
});
```

query can use index scan:

```text
IXSCAN
```

---

## 4. Unique index

```js
userSchema.index(
  {
    email: 1,
  },
  {
    unique: true,
  }
);
```

This enforces uniqueness at DB level.

Important:

```text
unique index
  !=
application validator
```

It protects against concurrency races.

---

## 5. Single-field index

Example:

```text
email
```

Useful for:

```text
find user by email
```

---

## 6. Compound index

Example:

```js
orderSchema.index({
  userId: 1,
  createdAt: -1,
});
```

Useful for:

```text
find orders for user
sorted newest first
```

---

## 7. Index order matters

Compound:

```text
(userId, status, createdAt)
```

can help query:

```text
userId = ?
status = ?
ORDER BY createdAt
```

But order is important.

Think in terms of your actual filters/sorts.

---

## 8. Prefix principle

An index:

```text
(a, b, c)
```

can commonly support prefixes such as:

```text
a
a + b
a + b + c
```

But not every query on:

```text
b
```

alone will benefit the same way.

This is a critical interview concept.

---

## 9. Equality, Sort, Range heuristic

A useful MongoDB index design heuristic:

```text
Equality
Sort
Range
```

Example query:

```text
userId = ?
status = ?
ORDER BY createdAt DESC
price > ?
```

Think about compound index order around equality fields first, then sort, then range depending on actual query needs.

Always verify with `explain()`.

---

## 10. Sorting with indexes

Query:

```js
Order.find({
  userId,
})
.sort({
  createdAt: -1,
});
```

Index:

```text
(userId, createdAt)
```

can help both filter and sort.

---

## 11. Pagination with indexes

Cursor pagination:

```text
createdAt DESC, _id DESC
```

Index:

```text
(createdAt, _id)
```

helps efficient continuation.

---

## 12. Low-cardinality indexes

Indexing:

```text
isActive: true/false
```

alone may not be very selective.

If half the collection matches, index may provide limited benefit.

Selectivity matters.

---

## 13. Partial indexes

Index only documents that match a condition.

Example concept:

```text
index email
only where deletedAt does not exist
```

Useful with soft deletion or sparse subsets.

---

## 14. Sparse indexes

Sparse indexes include only documents where indexed field exists.

Useful in some optional-field designs.

Understand how this interacts with uniqueness.

---

## 15. TTL indexes

MongoDB supports TTL indexes for automatic expiration.

Useful for:

- sessions
- OTPs
- temporary tokens
- cache-like documents

Example concept:

```text
expiresAt
   |
   v
TTL index
   |
   v
MongoDB deletes expired document later
```

Expiration is not guaranteed to happen at the exact millisecond.

---

## 16. Text indexes

MongoDB supports text search indexes.

Useful for basic full-text search.

But advanced search requirements may need MongoDB Atlas Search or dedicated search systems.

---

## 17. Multikey indexes

When indexing array fields, MongoDB can create multikey indexes.

Example:

```json
{
  "tags": [
    "node",
    "backend"
  ]
}
```

Index:

```text
tags
```

helps queries on array elements.

There are restrictions around compound multikey scenarios.

---

## 18. Indexes cost writes

Every insert/update may need index maintenance.

More indexes mean:

- slower writes
- more disk
- more memory
- more replication work

So:

```text
more indexes != always better
```

---

## 19. Indexes consume memory

Frequently used indexes benefit from fitting in working memory.

Huge unnecessary indexes increase cache pressure.

---

## 20. explain()

Use:

```js
db.users
  .find({
    email:
      "vikash@example.com",
  })
  .explain(
    "executionStats"
  );
```

Look at:

- winning plan
- COLLSCAN vs IXSCAN
- documents examined
- keys examined
- returned count
- execution time

---

## 21. Bad query signal

Example:

```text
documents examined = 1,000,000
documents returned = 10
```

That is a strong signal the query/index strategy needs attention.

---

## 22. Covered query

If all requested fields exist in the index, MongoDB may satisfy query without fetching full documents.

Example concept:

```text
index:
(email, name)

query:
filter email
select name
```

This can be highly efficient.

---

## 23. Regex and indexes

Prefix regex:

```text
^node
```

may use index better than:

```text
.*node.*
```

Substring search often cannot use normal B-tree-style indexes efficiently.

---

## 24. Index intersection

MongoDB can sometimes combine multiple indexes.

But a well-designed compound index often performs better for known frequent query patterns.

Do not depend on intersection blindly.

---

## 25. Indexing relationships

If Order references User:

```text
userId
```

and you frequently query:

```text
orders by user
```

index:

```text
userId
```

or:

```text
userId + createdAt
```

depending on sort.

---

## 26. Soft delete index

Common query:

```text
WHERE deletedAt = null
AND email = ?
```

A partial unique index on active documents may fit better than a simple unique index depending on data model.

---

## 27. Index migrations

Do not casually create large production indexes at app startup.

Large index builds can affect production.

Use controlled migrations/deployment procedures.

---

## 28. autoIndex caution

Mongoose may create indexes automatically depending on configuration.

In production, many teams prefer explicit controlled index migrations instead of automatic startup index creation.

---

## 29. Common mistakes

### Mistake 1
No indexes on frequent filters.

### Mistake 2
Index every field.

### Mistake 3
Wrong compound order.

### Mistake 4
Ignoring sort pattern.

### Mistake 5
Never using explain.

### Mistake 6
Auto-building heavy indexes unexpectedly in production.

---

## 30. Interview questions

### Why do indexes speed up reads?

They provide an ordered/searchable structure that avoids scanning the entire collection.

### What is the cost of indexes?

Disk, memory, and slower writes due to index maintenance.

### Why does compound index order matter?

The leading fields determine which query patterns and sort orders the index can efficiently support.

### What is a unique index?

A DB-level constraint ensuring indexed key values are unique.

### How do you know if an index works?

Use `explain()` and inspect query execution statistics.

---

## 31. Strong interview answer

> I design indexes from real query patterns rather than indexing every field. For MongoDB, I look at equality filters, sort order, and range predicates when choosing compound indexes, and I always verify with explain() rather than assuming an index is being used. I also account for write cost, memory, selectivity, and index-build strategy in production. Unique indexes are especially important because they enforce invariants safely under concurrent requests.

---

## Interview-Ready Summary

```text
Index
   |
   +--> faster reads
   +--> supports sort
   +--> supports uniqueness
   +--> supports pagination

Costs:
write overhead
disk
memory

Must know:
single
compound
unique
partial
sparse
TTL
text
multikey
explain()
```

## Section 10 Progress Map

```text
Database Architecture
      |
      v
MongoDB Integration
      |
      v
Mongoose
      |
      v
Schemas + Models
      |
      v
Relationships + populate
      |
      v
Indexing
      |
      v
Next:
Aggregation
Transactions
PostgreSQL
Connection Pooling
ORM vs Query Builder
Prisma / Drizzle
N+1
```

## Practice Task

Given these queries:

1. find user by email
2. list user orders newest first
3. list paid orders newest first
4. cursor-paginate products by createdAt + id
5. expire OTP documents automatically

Design the indexes and explain why.
