# Lesson 107 — MongoDB Integration

## Why MongoDB integration matters

MongoDB is common in Node.js applications because both work naturally with JSON-like documents.

But production MongoDB integration requires more than:

```js
mongoose.connect(...)
```

You should understand:

- connection lifecycle
- MongoDB driver vs Mongoose
- ObjectId
- document modeling
- projection
- pagination
- atomic operations
- transactions
- indexes

---

## 1. MongoDB mental model

```text
Database
   |
   v
Collections
   |
   v
Documents
```

Example document:

```json
{
  "_id": "...",
  "name": "Vikash",
  "email": "vikash@example.com"
}
```

---

## 2. MongoDB driver vs Mongoose

Official MongoDB driver:

```text
lower-level
direct MongoDB API
more control
```

Mongoose:

```text
ODM
schemas
models
validation
middleware
populate
```

Mongoose sits on top of the MongoDB driver.

---

## 3. Direct driver example

```js
import {
  MongoClient,
} from "mongodb";

const client =
  new MongoClient(
    process.env.MONGODB_URI
  );

await client.connect();

const db =
  client.db("app");

const users =
  db.collection("users");
```

---

## 4. Shared client

Create the Mongo client once for your application process.

Bad:

```js
router.get(
  "/users",
  async () => {
    const client =
      new MongoClient(uri);

    await client.connect();
  }
);
```

Do not open a new client for every request.

---

## 5. Connection pool

The MongoDB driver maintains a pool of network connections.

Conceptually:

```text
MongoClient
    |
    v
connection pool
    |
    +--> MongoDB
    +--> MongoDB
    +--> MongoDB
```

Application queries borrow connections as needed.

---

## 6. ObjectId

MongoDB commonly uses:

```text
ObjectId
```

as `_id`.

Example:

```js
import {
  ObjectId,
} from "mongodb";

const id =
  new ObjectId(
    req.params.id
  );
```

Validate before constructing/using invalid IDs.

---

## 7. Find one

```js
const user =
  await users.findOne({
    _id: id,
  });
```

---

## 8. Insert

```js
const result =
  await users.insertOne({
    name: "Vikash",
    email:
      "vikash@example.com",
  });
```

---

## 9. Update

```js
await users.updateOne(
  {
    _id: id,
  },
  {
    $set: {
      name: "New Name",
    },
  }
);
```

---

## 10. Delete

```js
await users.deleteOne({
  _id: id,
});
```

---

## 11. Projection

Avoid fetching unnecessary fields.

```js
await users.findOne(
  {
    _id: id,
  },
  {
    projection: {
      name: 1,
      email: 1,
    },
  }
);
```

This is especially important for sensitive fields.

---

## 12. Never expose password hashes

Even if password hashes are not plaintext, do not include them in normal API responses.

Use:
- projection
- serializers
- schema select rules

Defense in depth is better.

---

## 13. Atomic update operators

MongoDB supports atomic updates to individual documents.

Example inventory decrement:

```js
const product =
  await products.findOneAndUpdate(
    {
      _id: productId,
      stock: {
        $gte: quantity,
      },
    },
    {
      $inc: {
        stock: -quantity,
      },
    },
    {
      returnDocument: "after",
    }
  );
```

This can prevent overselling better than:

```text
read stock
   |
   v
check in Node
   |
   v
update later
```

which has a race condition.

---

## 14. Race condition example

Bad:

```js
const product =
  await products.findOne({
    _id: id,
  });

if (
  product.stock >= quantity
) {
  await products.updateOne(
    { _id: id },
    {
      $inc: {
        stock: -quantity,
      },
    }
  );
}
```

Two requests can both pass the stock check.

Prefer atomic conditional update.

---

## 15. Upsert

```js
await collection.updateOne(
  {
    email,
  },
  {
    $set: data,
  },
  {
    upsert: true,
  }
);
```

Use carefully.

Upsert semantics should match business rules.

---

## 16. Pagination

Offset style:

```js
collection
  .find(filter)
  .sort({
    createdAt: -1,
  })
  .skip(offset)
  .limit(limit);
```

For large datasets, cursor/keyset patterns are often better.

---

## 17. Cursor-style pagination

Example:

```js
collection.find({
  _id: {
    $lt: lastId,
  },
})
.sort({
  _id: -1,
})
.limit(20);
```

Works well when ordering aligns with `_id` semantics, but choose cursor field intentionally.

---

## 18. Aggregation

MongoDB aggregation pipeline:

```text
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

Example:

```js
orders.aggregate([
  {
    $match: {
      status: "paid",
    },
  },
  {
    $group: {
      _id: "$userId",
      total: {
        $sum: "$amount",
      },
    },
  },
]);
```

---

## 19. Transactions

MongoDB supports multi-document transactions in appropriate deployments.

Use transactions when multiple writes must commit/rollback together.

Example:

```text
create order
decrease inventory
create payment record
```

But do not use transactions unnecessarily for operations that can be modeled atomically in one document.

---

## 20. Embedding vs referencing

Embedding:

```json
{
  "orderId": "...",
  "items": [
    {
      "productId": "...",
      "name": "...",
      "price": 1000
    }
  ]
}
```

Referencing:

```json
{
  "userId": "..."
}
```

MongoDB modeling should follow access patterns.

---

## 21. Schema design starts from queries

Ask:

- how will data be read?
- how often updated?
- does child data belong strongly to parent?
- how large can arrays grow?
- do entities need independent lifecycle?

MongoDB is not "SQL tables but JSON."

---

## 22. Unbounded arrays

Danger:

```json
{
  "userId": "...",
  "notifications": [
    "... millions ..."
  ]
}
```

Documents have size limits and huge arrays become difficult to update/query.

Use separate collections for unbounded growth.

---

## 23. Indexes

Without indexes, filters may scan the entire collection.

Example:

```text
find user by email
```

should usually have:

```text
email index
```

More in Lesson 111.

---

## 24. NoSQL injection

Dangerous pattern:

```js
User.findOne({
  email:
    req.body.email,
});
```

if arbitrary operator-shaped objects are allowed.

Validate types strictly.

Do not allow client-provided Mongo operators.

---

## 25. Common mistakes

### Mistake 1
New MongoClient per request.

### Mistake 2
Unbounded arrays.

### Mistake 3
No indexes.

### Mistake 4
Read-then-write race conditions.

### Mistake 5
Returning raw DB documents.

### Mistake 6
Treating MongoDB as schema-free chaos.

---

## 26. Interview questions

### MongoDB driver vs Mongoose?

Driver gives direct low-level MongoDB access; Mongoose adds ODM features such as schemas, models, validation, middleware, and populate.

### Why shared MongoClient?

The client owns connection pooling and should be reused.

### How do you prevent inventory race conditions?

Use atomic conditional update operators instead of separate read/check/update steps.

### Embedding vs referencing?

Choose based on ownership, access patterns, update frequency, and document growth.

---

## 27. Strong interview answer

> In Node.js I create one shared MongoDB client or Mongoose connection for the application process and let the driver manage connection pooling. I design documents around access patterns, use projection to avoid overfetching, atomic update operators to avoid read-modify-write races, and indexes for common queries. I embed tightly owned bounded data and reference entities with independent lifecycles or unbounded growth.

---

## Interview-Ready Summary

```text
Node.js
   |
   v
MongoClient / Mongoose
   |
   v
Connection Pool
   |
   v
MongoDB

Think about:
atomic updates
embedding vs references
indexes
projection
transactions
pagination
```

## Practice Task

Design MongoDB collections for:

- users
- products
- orders
- chats

Explain where you would embed and where you would reference.
