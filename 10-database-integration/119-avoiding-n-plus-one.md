# Lesson 119 — Avoiding N+1 Queries

## What is the N+1 query problem?

Suppose you fetch:

```text
100 orders
```

Query 1:

```text
SELECT orders
```

Then for each order:

```text
query user
query user
query user
...
100 times
```

Total:

```text
1 + 100 queries
```

This is the N+1 problem.

## Why it is bad

It causes:

- high DB load
- network overhead
- latency
- poor scalability

## Solutions

### SQL joins

```sql
SELECT ...
FROM orders
JOIN users
ON orders.user_id = users.id;
```

### Batch query

Fetch all needed user IDs:

```text
WHERE id IN (...)
```

### ORM eager loading

Use relation-loading features where appropriate.

### MongoDB

Use:
- controlled `populate()`
- aggregation `$lookup`
- batch queries
- embedding where appropriate

## Be careful

Eager loading everything can create the opposite problem:

```text
one huge query
+
massive payload
```

Fetch only what is needed.

## Interview answer

> N+1 happens when one initial query is followed by one additional query per returned record. I avoid it using joins, batched queries, controlled eager loading, populate/$lookup where appropriate, or better data modeling.

## Quick Summary

```text
Bad:
1 query
+
N related queries

Better:
join
batch
eager load
better modeling
```

## Section 10 Final Map

```text
Database Architecture
      |
      +--> MongoDB
      +--> Mongoose
      +--> Schemas
      +--> Relationships
      +--> Indexes
      +--> Aggregation
      +--> Transactions
      |
      +--> PostgreSQL
      +--> Connection Pooling
      +--> ORM / Query Builder
      +--> Prisma / Drizzle
      +--> N+1 Queries
```
