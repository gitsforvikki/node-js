# Lesson 113 — MongoDB Transactions

## What is a MongoDB transaction?

A transaction groups multiple database operations so they either:

```text
all succeed
or
all fail
```

This gives atomic behavior across multiple operations.

## Example use case

Creating an order:

```text
decrease stock
   |
   v
create order
   |
   v
create payment record
```

If one step fails, the transaction can roll back the others.

## Basic Mongoose idea

```js
const session =
  await mongoose.startSession();

await session.withTransaction(
  async () => {
    await Product.updateOne(
      { _id: productId },
      {
        $inc: {
          stock: -1,
        },
      },
      { session }
    );

    await Order.create(
      [orderData],
      { session }
    );
  }
);

await session.endSession();
```

## Important point

Do not use a transaction when a single atomic MongoDB update can solve the problem.

Example:

```js
$inc
```

on one document may be simpler and faster.

## Common mistake

Starting transactions for every request.

Transactions add coordination and overhead.

## Interview answer

> MongoDB transactions are useful when multiple document operations must commit or roll back together. I use them for multi-step consistency requirements, but prefer single-document atomic operations when possible because they are simpler and cheaper.

## Quick Summary

```text
Transaction
   |
   +--> commit
   +--> rollback
   +--> multi-operation consistency
```
