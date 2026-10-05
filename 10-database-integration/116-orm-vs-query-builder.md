# Lesson 116 — ORM vs Query Builder

## ORM

An ORM maps database structures to application objects/models.

Examples:

- Prisma
- Sequelize
- TypeORM

Conceptually:

```js
userRepository.findMany(...)
```

instead of writing raw SQL.

## Query Builder

A query builder helps construct SQL programmatically while keeping SQL concepts visible.

Examples:

- Knex
- Drizzle

Conceptually:

```js
db
  .select()
  .from(users)
  .where(...)
```

## ORM advantages

- productivity
- model abstraction
- relations
- migrations
- type support

## ORM drawbacks

- hidden SQL
- abstraction surprises
- harder optimization in complex queries

## Query builder advantages

- more SQL control
- predictable queries
- easier optimization
- usually thinner abstraction

## Query builder drawbacks

- more database knowledge required
- more query code

## Raw SQL still matters

Even with an ORM, you should understand:

- joins
- indexes
- transactions
- query plans

## Interview answer

> ORMs provide a higher-level model abstraction and are productive for common CRUD operations. Query builders keep developers closer to SQL and provide more control. I choose based on application complexity, team knowledge, and how much query-level control is needed.

## Quick Summary

```text
ORM
  -> higher abstraction

Query Builder
  -> closer to SQL

Both still require
database fundamentals
```
