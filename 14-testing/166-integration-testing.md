# Lesson 166 — Integration Testing

## 1. Why do we need integration tests?
A unit test may prove a function works, but real bugs often happen **between components**: Express receives JSON, validation runs, a controller calls Mongoose, and a response is created. An integration test checks these pieces working **together**.

## 2. Unit vs integration
- Unit: `calculatePrice(100, 2)` gives 200, no database.
- Integration: `POST /products` inserts a product into a **test database**, then `GET /products` returns it.

## 3. Realistic architecture
```text
Supertest
   -> Express app (no listening port needed)
   -> middleware + route/controller
   -> Mongoose model
   -> isolated TEST MongoDB
   -> HTTP response assertions
```

## 4. Separate app from server startup
**src/app.js**
```js
import express from "express";
export const app = express();
app.use(express.json());

export function registerProductRoutes(Product) {
  app.post("/products", async (req, res) => {
    if (typeof req.body.name !== "string" || !req.body.name.trim()) {
      return res.status(400).json({ message: "Name required" });
    }
    const product = await Product.create({ name: req.body.name });
    res.status(201).json({ id: product.id, name: product.name });
  });
  app.get("/products", async (req, res) => {
    res.json(await Product.find().select("name").lean());
  });
}
```
**src/server.js** connects production DB and calls `app.listen()`. Tests import `app` without opening a permanent server.

## 5. Integration test with isolated MongoDB
Install:
```bash
npm install -D supertest mongodb-memory-server
npm install mongoose
```

Example test (Node built-in runner, ESM):
```js
import { before, after, beforeEach, test } from "node:test";
import assert from "node:assert/strict";
import request from "supertest";
import mongoose from "mongoose";
import { MongoMemoryServer } from "mongodb-memory-server";
import { app, registerProductRoutes } from "../src/app.js";

const productSchema = new mongoose.Schema({ name: { type: String, required: true } });
const Product = mongoose.model("Product", productSchema);
registerProductRoutes(Product);

let mongo;
before(async () => {
  mongo = await MongoMemoryServer.create();
  await mongoose.connect(mongo.getUri());
});
beforeEach(async () => { await Product.deleteMany({}); });
after(async () => {
  await mongoose.disconnect();
  await mongo?.stop();
});

test("POST then GET persists a product", async () => {
  const created = await request(app).post("/products").send({ name: "Mouse" });
  assert.equal(created.status, 201);
  const fetched = await request(app).get("/products");
  assert.equal(fetched.status, 200);
  assert.equal(fetched.body.length, 1);
  assert.equal(fetched.body[0].name, "Mouse");
});
```

**Note:** `mongodb-memory-server` downloads a MongoDB binary and may not work in offline/restricted environments. An isolated Docker MongoDB test container is another option. Do not use your real production URI.

## 6. Isolation and cleanup
Each test must start from predictable data. Clear collections, use fresh DBs, or wrap suitable operations in transactions with appropriate MongoDB topology. Transactions require replica-set support and have limitations.

## 7. What should integration tests verify?
Database validation, indexes/duplicate errors, serialization, relationships, controllers + middleware, transactional logic, and network/provider adapters using safe test dependencies.

## 8. Avoid flaky tests
Await all asynchronous setup and teardown. Do not reuse production data, depend on test order, or call live services unexpectedly. For parallel tests, isolate DB names and ports.

## 9. Practice
Add a test that rejects missing product names (400), one that writes two products, and one that verifies the database is clean between tests.

**Interview answer:** Integration tests check that multiple real application components work together, especially routing, middleware, database and serialization. They are slower than unit tests but catch connection and contract failures mocks may miss.
