# Lesson 112 — Aggregation

## What is aggregation?

Aggregation is used when you need to transform, group, summarize, or calculate data inside the database.

MongoDB uses an aggregation pipeline.

```text
Documents
   |
   v
$match
   |
   v
$group
   |
   v
$sort
   |
   v
$project
```

## Example

Total paid order amount per user:

```js
Order.aggregate([
  {
    $match: {
      status: "paid",
    },
  },
  {
    $group: {
      _id: "$userId",
      totalAmount: {
        $sum: "$amount",
      },
    },
  },
]);
```

## Important stages

- `$match` — filter documents
- `$group` — aggregate values
- `$sort` — sort output
- `$project` — reshape fields
- `$lookup` — combine collections
- `$unwind` — expand arrays

## Best practice

Filter early.

```text
$match
   |
   v
smaller dataset
   |
   v
expensive stages
```

This usually improves performance.

## Common mistake

Doing large calculations in Node.js after fetching thousands of documents.

Prefer database aggregation when the database can do it efficiently.

## Interview answer

> MongoDB aggregation processes documents through a pipeline of stages such as match, group, sort, project, and lookup. I use it for reporting and data transformation when the calculation is better handled close to the data.

## Quick Summary

```text
Aggregation
  -> filter
  -> group
  -> calculate
  -> reshape
  -> report
```
