# Lesson 117 — Prisma / Drizzle Concepts

## Prisma

Prisma is a TypeScript-friendly ORM/data toolkit.

Typical flow:

```text
Prisma Schema
    |
    v
Generated Client
    |
    v
Database
```

Example:

```js
const users =
  await prisma.user.findMany();
```

Strengths:

- strong developer experience
- generated client
- type safety
- schema-driven workflow

## Drizzle

Drizzle is a TypeScript ORM/query layer that stays closer to SQL.

Example:

```js
const result =
  await db
    .select()
    .from(users);
```

Strengths:

- SQL-like API
- strong TypeScript support
- lightweight abstraction
- explicit schema definitions

## High-level comparison

```text
Prisma
  -> more abstraction
  -> generated client

Drizzle
  -> closer to SQL
  -> explicit query builder style
```

## Important point

Neither tool removes the need to understand:

- indexes
- transactions
- joins
- normalization
- query performance

## Interview answer

> Prisma gives a higher-level schema-driven ORM experience with a generated typed client, while Drizzle stays closer to SQL with a strongly typed query-builder style. The choice depends on how much abstraction and SQL control the project wants.

## Quick Summary

```text
Prisma
  -> schema-driven ORM

Drizzle
  -> typed SQL-like ORM/query builder
```
