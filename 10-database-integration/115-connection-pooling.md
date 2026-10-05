# Lesson 115 — Connection Pooling

## What is connection pooling?

A connection pool keeps a reusable set of database connections.

```text
Node.js App
    |
    v
Connection Pool
    |
    +--> connection 1
    +--> connection 2
    +--> connection 3
    |
    v
Database
```

## Why use a pool?

Opening a new DB connection for every request is expensive.

A pool:

- reuses connections
- limits concurrency
- reduces connection overhead
- protects the database

## Example

```js
const pool =
  new Pool({
    connectionString:
      process.env.DATABASE_URL,
    max: 10,
  });
```

## Pool saturation

If:

```text
100 requests
10 DB connections
```

some requests wait for a free connection.

This is normal.

## Bigger pool is not always better

Suppose:

```text
20 app instances
x 50 connections
= 1000 DB connections
```

That may overwhelm the database.

## Common mistake

Increasing pool size to fix slow queries.

Often the real problem is:
- missing index
- expensive query
- too much concurrency

## Interview answer

> Connection pooling reuses a limited number of database connections instead of opening one per request. Pool size should be chosen based on total application instances, database capacity, and workload rather than simply making it very large.

## Quick Summary

```text
Pool
  -> reuse connections
  -> control concurrency
  -> reduce overhead
  -> protect DB
```
