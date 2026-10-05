# Lesson 118 — Database Transactions

## What is a transaction?

A transaction groups operations into one logical unit.

```text
BEGIN
  operation A
  operation B
  operation C
COMMIT
```

If something fails:

```text
ROLLBACK
```

## ACID

Transactions are associated with:

```text
A -> Atomicity
C -> Consistency
I -> Isolation
D -> Durability
```

## Example

Bank transfer:

```text
subtract ₹1000 from A
        |
        v
add ₹1000 to B
```

Both must succeed together.

## PostgreSQL example

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

COMMIT;
```

## In Node.js

Usually:

```text
get dedicated connection
   |
   v
BEGIN
   |
   v
queries
   |
   +--> COMMIT
   |
   +--> ROLLBACK on error
```

## Important point

Transactions should usually match a business consistency boundary.

Do not hold them open while:
- calling slow external APIs
- waiting for user input
- doing unnecessary work

## Interview answer

> A transaction makes a set of database operations behave as one unit. I use transactions when multiple writes must maintain a shared invariant, and I keep transactions as short as possible to reduce lock and concurrency impact.

## Quick Summary

```text
Transaction
   |
   +--> BEGIN
   +--> COMMIT
   +--> ROLLBACK
   +--> ACID
```
