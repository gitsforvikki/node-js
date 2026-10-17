# Lesson 171 — Database Testing with MongoDB and Mongoose

## 1. Why test the database?

Unit tests may prove validation logic works but miss real MongoDB issues:
- A schema rejects required fields.
- A unique index prevents duplicate emails.
- A query returns the wrong records.
- A middleware hook behaves differently for `save()` vs `findOneAndUpdate()`.
- A transaction fails under concurrency.

**Database tests** verify that the real database driver, schema, models, and queries behave correctly.

## 2. Never test against production

```text
Test runner
  -> separate test environment
  -> MongoDB test instance / container
  -> seeded test data
  -> assertions
  -> cleanup
```

A **separate database** is essential. Never put a production URI in test configuration. For CI use isolated containers or test-specific deployments.

## 3. Option A: MongoDB Memory Server

Install:

```bash
npm install mongoose
npm install -D mongodb-memory-server
```

This package starts a temporary MongoDB process for tests. Its first run may download a MongoDB binary. It may fail in restricted/offline environments and does not replace testing your production MongoDB topology.

**test/user.db.test.js**

```js
import { before, after, beforeEach, test } from "node:test";
import assert from "node:assert/strict";
import mongoose from "mongoose";
import { MongoMemoryServer } from "mongodb-memory-server";

const schema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  age: { type: Number, min: 18 },
});

const User = mongoose.model("TestUser", schema);
let mongo;

before(async () => {
  mongo = await MongoMemoryServer.create();
  await mongoose.connect(mongo.getUri());
  await User.init(); // Build declared indexes before duplicate tests
});

beforeEach(async () => {
  await User.deleteMany({});
});

after(async () => {
  await mongoose.disconnect();
  await mongo?.stop();
});

test("saves and reads a user", async () => {
  await User.create({ email: "a@example.com", age: 22 });
  const user = await User.findOne({ email: "a@example.com" });
  assert.equal(user.age, 22);
});

test("rejects invalid age", async () => {
  await assert.rejects(
    User.create({ email: "b@example.com", age: 12 }),
    (err) => err.name === "ValidationError"
  );
});

test("unique index prevents duplicate emails", async () => {
  await User.create({ email: "a@example.com", age: 22 });
  await assert.rejects(
    User.create({ email: "a@example.com", age: 24 }),
    (err) => err.code === 11000
  );
});
```

Run `node --test test/user.db.test.js`. Use `"type": "module"` in `package.json`.

## 4. Important: unique is NOT a validator

`unique: true` declares a **MongoDB unique index**. Duplicate documents cause a MongoDB duplicate-key error (usually `11000`), **not** a Mongoose `ValidationError`.

This is why database integration tests catch behavior that unit tests miss.

## 5. Option B: Docker test database

```bash
docker run --rm --name mongo-test -p 27018:27017 -d mongo:7
```

Use a test-only URI such as `mongodb://127.0.0.1:27018/shop_test`. Make sure you aren't colliding with another local service. Stop it with `docker stop mongo-test`.

For production-like behavior, test MongoDB server versions and replica-set features that match your deployment where practical.

## 6. Transactions and replica sets

MongoDB multi-document transactions require supported replica-set/sharded topology; a standalone test server is insufficient.

Test that:
1. Related writes commit together on success.
2. An error rolls back the transaction.
3. Conflicting or repeated operations follow your retry/idempotency design.

Use `MongoMemoryReplSet` (or a replica-set Docker deployment) when transaction tests need it.

## 7. Mongoose middleware test cases

```js
userSchema.pre("save", function () {
  this.email = this.email.toLowerCase();
});
```

Test `User.create({ email: "A@EXAMPLE.COM" })` stores lowercase; also test `findOneAndUpdate()` separately because save hooks do **not** run for query updates.

## 8. Isolation with parallel tests

If test files run concurrently, each should use independent databases/collections or carefully controlled cleanup. Dropping all collections from one test can break another test.

Prefer independent fixtures and cleanup after each test. Avoid shared ordering assumptions and flaky sleeps.

## 9. What to assert

- The inserted/updated document contains expected fields.
- Missing/invalid values are rejected.
- Database constraints and indexes work.
- Filters/sorting/pagination return correct results.
- Transactions do not leave partial changes on rollback.
- Sensitive fields are not accidentally returned.

## 10. Common mistakes and practice

Mistakes: production database in tests, failing to build unique indexes, assuming mocks prove real queries, ignoring replica-set requirements, and race conditions from shared cleanup.

Practice: test soft-delete filtering, an email uniqueness constraint, a `pre("save")` hook and pagination order.

**Interview answer:** Database testing validates the actual database contract—schemas, indexes, queries, hooks and transactions—using isolated test data and predictable setup/cleanup. For Mongoose, unique indexes and query middleware deserve explicit integration tests.
