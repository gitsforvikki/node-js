# Lesson 110 — Relationships and populate

## Why this lesson matters

MongoDB supports relationships even though it is not a relational database.

In Mongoose, references and `populate()` make relationships convenient.

But overusing populate can create:

- slow queries
- too many DB operations
- large response payloads
- hidden performance problems

---

## 1. Relationship types

Common relationships:

```text
one-to-one
one-to-many
many-to-many
```

Example:

```text
User -> Profile
User -> Orders
Users <-> Teams
```

---

## 2. One-to-many reference

Order schema:

```js
userId: {
  type:
    mongoose.Schema.Types.ObjectId,
  ref: "User",
  required: true,
}
```

Many orders can reference one user.

---

## 3. populate()

Query:

```js
const order =
  await Order
    .findById(id)
    .populate(
      "userId"
    );
```

Instead of only:

```json
{
  "userId": "..."
}
```

you receive the referenced user document.

---

## 4. Select fields while populating

```js
.populate(
  "userId",
  "name email"
)
```

This is much better than pulling every user field.

Never populate passwordHash accidentally.

---

## 5. Rename relationship fields clearly

A field called:

```text
userId
```

contains an ID before populate and a document after populate.

Some teams prefer:

```text
user
```

for references.

Be consistent and make types clear.

---

## 6. Nested populate

Example:

```js
Order
  .findById(id)
  .populate({
    path: "items.product",
    select: "name price",
  });
```

Can be convenient, but expensive.

---

## 7. populate is not SQL JOIN

Interview-important:

Mongoose populate is an ODM abstraction.

It may perform additional MongoDB queries or use population logic rather than behaving exactly like a relational SQL JOIN.

Do not explain it as simply "MongoDB join."

---

## 8. $lookup

MongoDB aggregation provides:

```text
$lookup
```

which performs server-side collection joining behavior.

Example concept:

```text
orders
  |
  v
$lookup users
  |
  v
combined documents
```

Different from ordinary populate abstraction.

---

## 9. N+1 problem

Imagine:

```text
100 orders
```

and each order causes independent related lookup.

Potential pattern:

```text
1 query for orders
+
100 related queries
```

This is N+1.

Mongoose may optimize some population patterns, but you must still inspect query behavior.

---

## 10. Over-population

Bad:

```js
Order.find()
  .populate("user")
  .populate("items.product")
  .populate("coupon")
  .populate("payment")
  .populate("shippingAddress");
```

This can turn one endpoint into a huge data graph.

Only fetch what the client needs.

---

## 11. Circular relationships

Example:

```text
User -> Team
Team -> Users
```

Avoid recursively populating indefinitely.

Define response boundaries.

---

## 12. Virtual populate

Mongoose supports virtual population.

Example concept:

User documents do not store order IDs.

Order stores:

```text
userId
```

User virtual can define:

```text
orders
```

based on foreign field.

This avoids maintaining large arrays of child references.

---

## 13. Parent arrays can become dangerous

Bad:

```json
{
  "_id": "user1",
  "orderIds": [
    "... millions ..."
  ]
}
```

Better:

```text
Order stores userId
```

Then query orders by userId.

This supports unbounded one-to-many growth.

---

## 14. Embed instead of populate when appropriate

Order items are often embedded snapshots.

Why?

You usually need them whenever loading order.

```text
Order
  |
  +--> items embedded
```

No need to populate each order item unless current product data is specifically required.

---

## 15. Historical consistency

If an order references Product only and then product price changes, old order display may become incorrect.

Store snapshot:

```json
{
  "productId": "...",
  "name": "Laptop",
  "price": 50000
}
```

Reference can still exist for navigation.

---

## 16. Many-to-many

Example users and teams.

Option:

```text
membership collection
```

```json
{
  "userId": "...",
  "teamId": "...",
  "role": "admin"
}
```

This is often cleaner when relationship itself has data.

---

## 17. Relationship entity pattern

If relationship has attributes:

```text
joinedAt
role
status
permissions
```

model it as its own collection/document.

This is similar to a join table in SQL.

---

## 18. populate + lean

You can combine:

```js
Order
  .find({})
  .populate(...)
  .lean();
```

Useful for read-only endpoints, but understand how your Mongoose version/plugins affect virtuals and transforms.

---

## 19. Performance measurement

Do not guess.

Use:
- MongoDB profiler
- query logs
- `explain()`
- application tracing

Measure whether populate is actually expensive.

---

## 20. Common mistakes

### Mistake 1
Populate everything.

### Mistake 2
Huge parent arrays of child IDs.

### Mistake 3
Using references where snapshots are needed.

### Mistake 4
Calling populate a SQL JOIN.

### Mistake 5
Ignoring N+1 behavior.

### Mistake 6
No field projection.

---

## 21. Interview questions

### What does populate do?

It resolves referenced ObjectIds into related Mongoose documents.

### Is populate same as SQL JOIN?

No. It is an ODM-level population abstraction and may execute additional queries.

### When should you embed instead?

When data is tightly owned, bounded, and usually read with parent.

### Why use virtual populate?

To query child documents by foreign key without storing an ever-growing child ID array on parent.

---

## 22. Strong interview answer

> In Mongoose, I use references for entities with independent lifecycles and populate only when the endpoint truly needs related data. I use field selection aggressively and avoid deep population graphs because they can hide significant query cost. For unbounded one-to-many relationships, I usually store the parent reference on the child and query by it instead of keeping a giant array on the parent. For historical records such as order items, I often embed a snapshot rather than depending entirely on live product data.

---

## Interview-Ready Summary

```text
Relationship design
   |
   +--> embed
   +--> reference
   +--> virtual populate
   +--> relationship collection

populate:
convenient
not free
not SQL JOIN

Watch:
N+1
payload size
deep graphs
```

## Practice Task

Design relationships for:

- User -> Orders
- Order -> Items
- Item -> Product
- User <-> Team

Decide where to:
- embed
- reference
- populate
- use relationship collection
