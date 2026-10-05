# Lesson 114 — PostgreSQL Integration

## Why PostgreSQL with Node.js?

PostgreSQL is a relational database commonly used with Node.js for applications requiring:

- strong relational modeling
- joins
- constraints
- transactions
- complex queries
- ACID guarantees

## Typical architecture

```text
Node.js
   |
   v
PostgreSQL Driver / ORM
   |
   v
Connection Pool
   |
   v
PostgreSQL
```

## Using pg

```js
import {
  Pool,
} from "pg";

const pool =
  new Pool({
    connectionString:
      process.env.DATABASE_URL,
  });
```

Query:

```js
const result =
  await pool.query(
    "SELECT id, name FROM users WHERE id = $1",
    [userId]
  );
```

## Parameterized queries

Never build SQL like:

```js
`SELECT * FROM users WHERE email = '${email}'`
```

Use parameters:

```sql
WHERE email = $1
```

This helps prevent SQL injection.

## PostgreSQL strengths

- foreign keys
- unique constraints
- transactions
- joins
- indexing
- advanced SQL
- JSON support

## Common mistake

Creating a new DB connection for every request.

Use a pool.

## Interview answer

> In Node.js I typically access PostgreSQL through a pooled driver such as pg, an ORM, or a query builder. I use parameterized queries, rely on database constraints for integrity, and keep connection management centralized.

## Quick Summary

```text
Node.js
   |
   v
Pool
   |
   v
PostgreSQL

Always:
parameterize queries
use constraints
reuse connections
```
