# Lesson 109 — Schemas and Models

## Why schema design matters

Schema design determines:

- data consistency
- validation behavior
- query efficiency
- index strategy
- maintainability

Even in MongoDB, schema design is critical.

"Schema flexible" does not mean "schema does not matter."

---

## 1. Schema vs Model

Schema:

```text
definition
```

Model:

```text
runtime interface to collection
```

Example:

```js
const schema =
  new mongoose.Schema({...});

const User =
  mongoose.model(
    "User",
    schema
  );
```

---

## 2. Field types

Common Mongoose types:

- String
- Number
- Boolean
- Date
- ObjectId
- Decimal128
- Buffer
- Array
- Map
- Mixed

Choose based on domain semantics.

---

## 3. Required fields

```js
email: {
  type: String,
  required: true,
}
```

Application should still validate requests earlier.

Schema validation is a second layer.

---

## 4. Enum

```js
status: {
  type: String,
  enum: [
    "pending",
    "paid",
    "cancelled",
  ],
}
```

Useful for bounded domain states.

---

## 5. Default

```js
role: {
  type: String,
  default: "user",
}
```

Do not allow client to override privileged defaults unless authorized.

---

## 6. Min/max

```js
quantity: {
  type: Number,
  min: 1,
}
```

---

## 7. String normalization

Example:

```js
email: {
  type: String,
  lowercase: true,
  trim: true,
}
```

But normalization rules should match your domain.

---

## 8. Timestamps

```js
{
  timestamps: true,
}
```

Common for:
- auditing
- sorting
- cursor pagination

---

## 9. Embedded subdocuments

Example order item:

```js
const orderItemSchema =
  new mongoose.Schema(
    {
      productId: {
        type:
          mongoose.Schema.Types.ObjectId,
        required: true,
      },

      name: String,
      price: Number,
      quantity: Number,
    },
    {
      _id: false,
    }
  );
```

Then:

```js
items: [
  orderItemSchema
]
```

---

## 10. Why embed snapshots in orders?

Product name/price can change later.

Order should preserve historical purchase state.

So order item may store:

```text
productId
name at purchase time
price at purchase time
quantity
```

This is denormalization for correctness/history.

---

## 11. References

Example:

```js
userId: {
  type:
    mongoose.Schema.Types.ObjectId,
  ref: "User",
  required: true,
}
```

Reference is useful when related entity has its own lifecycle.

---

## 12. Embed vs reference decision

Embed when:

- strongly owned by parent
- read together
- bounded size
- no independent lifecycle

Reference when:

- shared by many documents
- independent lifecycle
- large/unbounded
- updated independently

---

## 13. Mixed type caution

```js
metadata: {
  type:
    mongoose.Schema.Types.Mixed,
}
```

Flexible but weakens validation and predictability.

Use only when schema genuinely needs flexibility.

---

## 14. Maps

Useful for dynamic keys:

```js
preferences: {
  type: Map,
  of: String,
}
```

Better than fully unstructured objects when value type is known.

---

## 15. Money fields

Avoid floating-point money where precision matters.

Possible approach:

```text
amountInPaise: integer
currency: INR
```

Example:

```js
amount: {
  type: Number,
  min: 0,
}
```

with invariant:

```text
amount always stored in smallest currency unit
```

---

## 16. Soft delete

Possible fields:

```text
deletedAt
isDeleted
```

But soft delete affects every query.

You must ensure:
- normal reads exclude deleted rows
- unique constraints behave correctly
- admin restore semantics are clear

---

## 17. Schema methods vs service methods

Schema methods are okay for document-local behavior.

Example:

```text
compare password
full name
```

Business workflow:

```text
checkout
cancel subscription
refund order
```

belongs in services.

---

## 18. Validation vs business rules

Schema:

```text
quantity >= 1
```

Business:

```text
quantity <= available stock
```

Different layers.

---

## 19. Index declarations

Example:

```js
userSchema.index(
  {
    email: 1,
  },
  {
    unique: true,
  }
);
```

Composite:

```js
orderSchema.index({
  userId: 1,
  createdAt: -1,
});
```

---

## 20. Schema versioning

As applications evolve:

```text
old documents
new schema
```

You may need:
- migrations
- backward-compatible readers
- defaults
- background migration jobs

MongoDB does not eliminate migration needs.

---

## 21. Model naming

Mongoose commonly pluralizes model names into collection names.

Example:

```text
User model
  -> users collection
```

You can override collection name if needed.

---

## 22. Prevent model recompilation in dev/serverless

In hot-reload environments, repeated:

```js
mongoose.model(
  "User",
  schema
);
```

can cause model overwrite errors.

Common pattern:

```js
export const User =
  mongoose.models.User ||
  mongoose.model(
    "User",
    userSchema
  );
```

This is especially relevant in Next.js/serverless development.

---

## 23. Common mistakes

### Mistake 1
Everything as Mixed.

### Mistake 2
Huge embedded arrays.

### Mistake 3
Storing mutable product data only by reference in historical order records.

### Mistake 4
Business logic inside schema hooks.

### Mistake 5
No migration strategy.

### Mistake 6
Floating-point money without clear convention.

---

## 24. Interview questions

### Schema vs model?

Schema defines shape/behavior; Model interacts with the collection.

### Embed vs reference?

Embed bounded owned data read with parent; reference shared/independent entities.

### Why store order item snapshot?

To preserve historical price/name even if product later changes.

### Does MongoDB need schema migrations?

Yes, application schema evolution still requires handling old documents.

---

## 25. Strong interview answer

> I design MongoDB schemas around access patterns and data lifecycle. I embed data that is strongly owned, bounded, and commonly read with the parent, while referencing shared or independently changing entities. I keep domain invariants clear, use database indexes for uniqueness and query performance, avoid unbounded arrays, and preserve historical snapshots where mutable source data should not change past records.

---

## Interview-Ready Summary

```text
Schema
   |
   +--> fields
   +--> validation
   +--> defaults
   +--> indexes
   +--> embedded docs
   +--> references

Design around:
ownership
read patterns
growth
history
update frequency
```

## Practice Task

Design Mongoose schemas for:

- User
- Product
- Order
- OrderItem
- Payment

Explain every embed/reference decision.
