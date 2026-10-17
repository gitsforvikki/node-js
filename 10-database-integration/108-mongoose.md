# Lesson 108 — Mongoose

## What is Mongoose?

Mongoose is an Object Data Modeling (ODM) library for MongoDB and Node.js.

It adds structure on top of the MongoDB driver.

Key concepts:

- Schema
- Model
- Document
- Validation
- Middleware
- Query helpers
- Population

---

## 1. Mental model

```text
MongoDB
   |
   v
MongoDB Driver
   |
   v
Mongoose
   |
   v
Schema + Model
   |
   v
Application
```

---

## 2. Why use Mongoose?

Benefits:

- schema definitions
- validation
- defaults
- hooks
- virtuals
- populate
- model methods
- convenient querying

Trade-off:

- abstraction overhead
- hidden behavior if misunderstood
- ODM-specific patterns

---

## 3. Connect

```js
import mongoose
  from "mongoose";

await mongoose.connect(
  process.env.MONGODB_URI
);
```

Do this once at application startup.

---

## 4. Schema

```js
const userSchema =
  new mongoose.Schema({
    name: {
      type: String,
      required: true,
    },

    email: {
      type: String,
      required: true,
      unique: true,
    },
  });
```

---

## 5. Model

```js
const User =
  mongoose.model(
    "User",
    userSchema
  );
```

Model represents a collection interface.

---

## 6. Document

```js
const user =
  new User({
    name: "Vikash",
    email:
      "vikash@example.com",
  });
```

This is a Mongoose document instance.

---

## 7. Save

```js
await user.save();
```

Mongoose:
- validates
- runs middleware
- writes through driver

---

## 8. Querying

```js
const user =
  await User.findById(id);
```

```js
const users =
  await User.find({
    active: true,
  });
```

---

## 9. Queries are thenable

Mongoose queries behave promise-like.

Example:

```js
await User.findOne({
  email,
});
```

For explicit query execution:

```js
await User
  .findOne({
    email,
  })
  .exec();
```

This can make intent clearer.

---

## 10. Validation

Schema:

```js
age: {
  type: Number,
  min: 18,
  max: 100,
}
```

Mongoose validation helps protect model-level data.

But it does not replace:
- request validation
- DB unique constraints
- business rules

---

## 11. unique is not a validator

This is a famous interview point.

```js
email: {
  type: String,
  unique: true,
}
```

`unique` tells Mongoose to create/use a unique index.

It is not normal validation logic.

Concurrent requests can still race until the database unique index decides the winner.

---

## 12. Defaults

```js
status: {
  type: String,
  default: "active",
}
```

---

## 13. Timestamps

```js
new mongoose.Schema(
  {...},
  {
    timestamps: true,
  }
);
```

Adds:

```text
createdAt
updatedAt
```

---

## 14. select: false

Useful for sensitive fields:

```js
passwordHash: {
  type: String,
  required: true,
  select: false,
}
```

Normal queries exclude it.

When needed:

```js
User
  .findOne({ email })
  .select("+passwordHash");
```

---

## 15. Middleware hooks

**Mongoose middleware (hooks)** are functions that run **before or after a Mongoose operation**, such as saving or querying data.

- `schema.pre("save", fn)`: runs **before** a save.
- `schema.post("save", fn)`: runs **after** a successful save.

**Simple example — see the execution order:**

```js
import mongoose from "mongoose";

const userSchema = new mongoose.Schema({
  name: String,
});

userSchema.pre("save", function () {
  console.log("1. Before saving:", this.name);
});

userSchema.post("save", function (doc) {
  console.log("3. Saved:", doc.name);
});

// Define hooks BEFORE compiling the model.
const User = mongoose.model("User", userSchema);

console.log("0. Creating user");
await User.create({ name: "Vikash" });
console.log("4. Create finished");
```

Expected output after a successful database connection:

```text
0. Creating user
1. Before saving: Vikash
3. Saved: Vikash
4. Create finished
```

**Flow:** `User.create()` → `pre("save")` → MongoDB insert → `post("save")` → `create()` completes.

This is useful for small model-related tasks such as normalizing data, password hashing, and logging. Do **not** put complex payment or business workflows inside hooks.

Mongoose has **document, query, model, and aggregate** middleware. The next section explains document and query middleware separately.

---

## 16. Document middleware vs query middleware

Both use `pre()` or `post()`, but **what `this` refers to is different**.

### A. Document middleware

Runs for a document operation such as `user.save()` or `User.create()`. Here, `this` is the **document**, so you can access fields directly.

```js
const userSchema = new mongoose.Schema({
  name: String,
  email: String,
});

userSchema.pre("save", function () {
  console.log("Saving user:", this.name);
  this.email = this.email.toLowerCase();
});

const User = mongoose.model("User", userSchema);

await User.create({
  name: "Vikash",
  email: "VIKASH@EXAMPLE.COM",
});
```

The hook runs before the document is saved, and the stored email is lowercase. `save` middleware also runs when `User.create()` saves a document.

### B. Query middleware

Runs for queries such as `User.find()` or `User.findOneAndUpdate()`. Here, `this` is a **Mongoose query**, **not** an individual user document.

Example — exclude soft-deleted users from `find()` queries:

```js
const userSchema = new mongoose.Schema({
  name: String,
  isDeleted: { type: Boolean, default: false },
});

userSchema.pre("find", function () {
  this.where({ isDeleted: false });
});

const User = mongoose.model("User", userSchema);

const users = await User.find(); // excludes isDeleted: true
```

This hook changes only `find()` queries; it does not automatically apply to `findOne()` or every other read method. Add the relevant hooks if you want that behavior across other query methods.

### C. Important: save hooks do not run on query updates

```js
await user.save();                 // pre("save") runs
await User.create({ name: "A" }); // pre("save") runs

await User.findOneAndUpdate(
  { name: "A" },
  { $set: { name: "B" } }
); // pre("save") does NOT run
```

To react to that update, define a separate **query middleware** hook:

```js
userSchema.pre("findOneAndUpdate", function () {
  console.log("Updating with filter:", this.getQuery());
  console.log("Update data:", this.getUpdate());
});
```

Define hooks **before** `mongoose.model("User", userSchema)`. This query hook does not directly give you the document being updated.

**Password safety:** A password hash implemented only in `pre("save")` will **not** run when a password changes using `findOneAndUpdate()` or `findByIdAndUpdate()`. Handle password updates explicitly and safely.

| Document middleware | Query middleware |
| --- | --- |
| Runs on document operations | Runs on query operations |
| `this` is the document | `this` is the query |
| Example: `pre("save")` | Example: `pre("findOneAndUpdate")` |
| Access `this.name` | Access `this.getQuery()` / `this.getUpdate()` |

**Remember:** A hook only runs for the matching operation. Don't assume all update methods trigger `save` middleware.

---

## 17. Password hashing hook caveat

Common pattern:

```js
userSchema.pre(
  "save",
  async function () {
    if (
      !this.isModified(
        "password"
      )
    ) {
      return;
    }

    this.password =
      await hash(
        this.password
      );
  }
);
```

Without `isModified`, password could be rehashed on unrelated saves.

---

## 18. Instance methods

```js
userSchema.methods
  .comparePassword =
  function (candidate) {
    return verify(
      candidate,
      this.passwordHash
    );
  };
```

Use carefully to avoid bloated model classes.

---

## 19. Static methods

```js
userSchema.statics
  .findByEmail =
  function (email) {
    return this.findOne({
      email,
    });
  };
```

Repositories may be cleaner for larger systems.

---

## 20. Virtuals

Virtual field:

```js
userSchema.virtual(
  "fullName"
).get(function () {
  return `${this.firstName} ${this.lastName}`;
});
```

Not stored in DB.

---

## 21. lean()

```js
const users =
  await User
    .find({})
    .lean();
```

Returns plain JavaScript objects instead of full Mongoose documents.

Benefits:
- less memory
- faster reads

Trade-off:
- no document methods
- no document change tracking
- virtuals/getters behavior depends on configuration/plugins

Excellent interview topic.

---

## 22. Projection

```js
User
  .find({})
  .select(
    "name email"
  );
```

Avoid fetching fields you do not need.

---

## 23. Query chaining

```js
User
  .find({
    active: true,
  })
  .sort({
    createdAt: -1,
  })
  .limit(20);
```

Readable, but always understand generated query behavior.

---

## 24. findOneAndUpdate

```js
const user =
  await User.findOneAndUpdate(
    {
      _id: id,
    },
    {
      $set: {
        name,
      },
    },
    {
      new: true,
      runValidators: true,
    }
  );
```

Important:
- validation behavior may differ from save
- hooks differ
- use options intentionally

---

## 25. Mongoose is not a substitute for MongoDB knowledge

You still need to understand:

- indexes
- aggregation
- transactions
- document modeling
- query plans
- atomicity

ODM convenience cannot replace database fundamentals.

---

## 26. Common mistakes

### Mistake 1
Connecting on every request.

### Mistake 2
Thinking unique is validator.

### Mistake 3
No `lean()` for read-heavy paths.

### Mistake 4
Heavy business logic in hooks.

### Mistake 5
Assuming all update methods trigger same hooks.

### Mistake 6
Returning raw documents directly.

---

## 27. Interview questions

### What is Mongoose?

An ODM for MongoDB that provides schemas, models, validation, middleware, population, and higher-level query APIs.

### Schema vs Model?

Schema defines structure/behavior; Model is the interface used to query and create documents for a collection.

### What does lean do?

Returns plain objects instead of full Mongoose documents, reducing overhead for read-only queries.

### Is unique a validator?

No. It represents a unique index requirement.

---

## 28. Strong interview answer

> Mongoose is an ODM layer over the MongoDB driver. I use schemas to define document structure and validation, models to query collections, and features such as timestamps, selective fields, and populate where appropriate. I use lean for read-only queries that do not need document behavior and avoid putting core business workflows into hooks. I also remember that database concerns such as indexes, atomicity, and unique constraints still belong to MongoDB itself.

---

## Interview-Ready Summary

```text
Schema
  -> structure + rules

Model
  -> collection API

Document
  -> model instance

Mongoose adds:
validation
hooks
virtuals
populate
query helpers

Important:
lean()
unique != validator
```

## Practice Task

Create a Mongoose User model with:

- name
- email unique index
- passwordHash select false
- role enum
- timestamps
- comparePassword method
