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

#### Custom query method (query helper): `onlyActive()`

A **query helper** is different from query middleware: you define it using `schema.query` and **call it yourself** when building a query. You can call it in a controller or service, not only inside the schema file.

```js
// models/user.model.js — define BEFORE mongoose.model()
userSchema.query.onlyActive = function () {
  return this.where({ active: true });
};

// controllers/user.controller.js — use the exported User model
const users = await User.find().onlyActive();
// Same filter as User.find({ active: true })
```

**Flow:** `User.find()` creates a query → `onlyActive()` adds the filter → `await` executes the query. Here `this` is the query object. Unlike `pre("find")`, `onlyActive()` does **not** run automatically; it runs only when you call it. (The schema must also contain an `active` Boolean field.)

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

**What:** A custom function available on **one Mongoose document**. Use it when the behavior belongs to an individual user, such as checking their password.

```js
import bcrypt from "bcryptjs";

userSchema.methods.comparePassword = async function (candidate) {
  return bcrypt.compare(candidate, this.passwordHash);
};

const User = mongoose.model("User", userSchema);

// passwordHash is excluded by select: false in the schema.
const user = await User.findOne({ email }).select("+passwordHash");
if (!user || !(await user.comparePassword(inputPassword))) {
  throw new Error("Invalid credentials");
}
```

**How it works:** `user.comparePassword(...)` uses `this.passwordHash` from **that user document**. `bcrypt.compare()` checks the plaintext candidate against the stored hash; never compare plaintext passwords directly.

**Remember:** Instance methods are available on Mongoose documents, not on plain objects returned by `lean()`. Keep complex business workflows in services rather than adding everything to the model.

### Simple instance method: `isAdult()`

Define the method on the schema, then call it on any **Mongoose document**, including inside a controller or service.

```js
// models/user.model.js — define BEFORE mongoose.model()
const userSchema = new mongoose.Schema({
  name: String,
  age: Number,
});

userSchema.methods.isAdult = function () {
  return this.age >= 18;
};

const User = mongoose.model("User", userSchema);

const user1 = new User({ name: "Vikash", age: 24 });
const user2 = new User({ name: "Rahul", age: 17 });

console.log(user1.isAdult()); // true
console.log(user2.isAdult()); // false
```

**Why:** `this` refers to the document calling the method, so each user checks their own age. You can also fetch a user in a controller with `await User.findById(id)` and call `user.isAdult()` after checking that the user exists. Do not call it on a `lean()` result.

---

## 19. Static methods

**What:** A custom function available on the **Model itself**. Use it to reuse common collection queries.

```js
userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email });
};

const User = mongoose.model("User", userSchema);

const user = await User.findByEmail("vikash@example.com");
```

**Difference:** `User.findByEmail()` is a **static method** on the model; `user.comparePassword()` is an **instance method** on one document. In a larger app, repositories/services can be a clearer home for complex queries and business rules.

---

## 20. Virtuals

**What:** A property calculated from other fields, **not stored in MongoDB**. Useful for display-only values such as a full name.

```js
const userSchema = new mongoose.Schema(
  { firstName: String, lastName: String },
  { toJSON: { virtuals: true } }
);

userSchema.virtual("fullName").get(function () {
  return `${this.firstName} ${this.lastName}`;
});

const User = mongoose.model("User", userSchema);
const user = new User({ firstName: "Vikash", lastName: "Kumar" });

console.log(user.fullName); // "Vikash Kumar"
console.log(user.toJSON().fullName); // "Vikash Kumar"
```

**Why useful:** You avoid storing duplicate data (`fullName`) that might become outdated when `firstName` changes. Virtuals do not appear in normal `lean()` results without extra configuration/plugins.

---

## 21. lean()

**What:** By default, Mongoose converts query results into full **Mongoose documents** with methods, change tracking, and other features. `lean()` skips that conversion and returns **plain JavaScript objects**.

```js
const users = await User.find({ active: true })
  .select("name email")
  .lean();

console.log(users[0].name); // Works: normal object property
// users[0].save();         // Does NOT work: no document method
```

**When useful:** Read-only API endpoints, lists, and dashboards where you only need to send data. It generally uses less memory and reduces Mongoose processing.

**When not to use:** If you need `document.save()`, instance methods, or ordinary document virtuals/getters. `lean()` does not automatically add those features. It does **not** make MongoDB itself execute a query faster; it mainly reduces work in Node.js.

---

## 22. Projection

**What:** Projection means **selecting only the fields you need** from a MongoDB document.

```js
const users = await User.find({})
  .select("name email")
  .lean();

// Each result contains name, email, and usually _id.
```

**Why useful:** Less data travels from MongoDB to Node.js, and you avoid accidentally returning unnecessary fields (especially sensitive ones).

```js
const user = await User.findById(id).select("-email -phone");
// Excludes email and phone.
```

**Important:** `_id` is included by default; use `.select("name email -_id")` if you want to exclude it. Projection is not a replacement for proper authorization or response sanitization.

---

## 23. Query chaining

**What:** Build a query step by step using readable methods for **filtering, sorting, and limiting** results.

Example: show the 10 newest active users.

```js
const users = await User.find({ active: true })
  .sort({ createdAt: -1 })
  .limit(10)
  .select("name email createdAt")
  .lean();
```

**Execution meaning:**
1. `find({ active: true })`: only active users.
2. `sort({ createdAt: -1 })`: newest first (`1` means ascending).
3. `limit(10)`: at most 10 results.
4. `select(...)`: return only required fields.
5. `lean()`: plain objects for a read-only response.

**Why useful:** Cleaner API queries and controlled response sizes. For consistent ordering when timestamps tie, add `_id` as a second sort field. Index frequently used filter/sort fields for large collections.

---

## 24. findOneAndUpdate

**What:** Find one matching document and update it **in one MongoDB operation**, without fetching it first and then calling `save()`.

```js
const updatedUser = await User.findOneAndUpdate(
  { _id: id },                     // Which user?
  { $set: { name: "Vikash Kumar" } }, // What to change?
  {
    new: true,          // Return updated document
    runValidators: true // Run supported update validators
  }
);

if (!updatedUser) {
  throw new Error("User not found");
}
console.log(updatedUser.name); // "Vikash Kumar"
```

**Why useful:** Convenient for a single update, avoids a separate read-then-save flow, and MongoDB performs the single-document update atomically.

**Important differences from `save()`:**
- `pre("save")` / `post("save")` **do not run**; use `findOneAndUpdate` query middleware if needed.
- Update validation is **off by default**; `runValidators: true` enables supported validators, but update validation differs from full document validation.
- `new: true` returns the updated document; without it, the default is the pre-update document.
- A nonmatching filter returns `null`.

For password changes, do not assume a password-hashing `save` hook runs on this method.

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
