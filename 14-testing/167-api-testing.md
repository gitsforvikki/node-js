# Lesson 167 — API Testing

## 1. What is API testing?
API testing checks the **HTTP contract** between frontend and backend: method, URL, headers, status, response JSON, authentication and error format.

A controller can work internally and still return wrong JSON or an incorrect status. API tests catch those mistakes.

## 2. Example: products API contract
| Request | Expected result |
| --- | --- |
| GET /products | 200 + array |
| GET /products/:id | 200 + product, or 404 |
| POST /products (valid) | 201 + created product |
| POST /products (invalid) | 400 |
| DELETE /products/:id without login | 401 |

## 3. Test manually with curl
```bash
curl -i http://localhost:3000/products
curl -i -X POST http://localhost:3000/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Keyboard","price":999}'
```
Use Postman or Insomnia for manual exploration, but automate important regression tests.

## 4. Express 5 API example
```js
import express from "express";
export const app = express();
app.use(express.json());

app.post("/products", (req, res) => {
  const { name, price } = req.body;
  if (typeof name !== "string" || !name.trim() ||
      typeof price !== "number" || !Number.isFinite(price) || price < 0) {
    return res.status(400).json({ code: "INVALID_PRODUCT" });
  }
  res.status(201).json({ id: "p1", name, price });
});
```

## 5. Automated API test with Supertest
```bash
npm install -D supertest
```
```js
import test from "node:test";
import assert from "node:assert/strict";
import request from "supertest";
import { app } from "../src/app.js";

test("valid product returns 201 and JSON", async () => {
  const res = await request(app).post("/products")
    .send({ name: "Keyboard", price: 999 });
  assert.equal(res.status, 201);
  assert.match(res.headers["content-type"], /json/);
  assert.equal(res.body.name, "Keyboard");
});

test("bad input returns 400", async () => {
  const res = await request(app).post("/products")
    .send({ name: "", price: -10 });
  assert.equal(res.status, 400);
  assert.equal(res.body.code, "INVALID_PRODUCT");
});
```

## 6. Essential API scenarios
**Happy path:** request succeeds. **Negative path:** invalid input, forbidden access, missing resource. **Edge case:** duplicate email, long input, empty arrays, unusual characters. **Failure path:** database unavailable, safe 500/503.

For authenticated APIs, check 401 vs 403 and resource ownership; hiding buttons in React is not an API permission check.

## 7. Idempotency and HTTP semantics
GET should not change business data. Repeating PUT should have the same intended effect. Repeating a payment POST may cause duplicate charges unless an idempotency mechanism is used; test it explicitly.

## 8. Contract and schema validation
Verify stable response field names and types. Consider OpenAPI to document request/response schemas and build contract tests. Unit, API, and integration are **overlapping dimensions**: an API test might be integration-level if it uses a real database.

## 9. Common mistakes
Checking status only; hardcoding live API keys; testing against production data; not awaiting requests; ignoring negative cases; making tests depend on each other.

## 10. Practice
Write API tests for GET /products, POST /products invalid payload, 404 and unauthorized DELETE. Verify JSON error shape too.

**Interview answer:** API testing validates HTTP contracts—status, body, headers, auth and errors—through real HTTP-like requests. Supertest automates Express API tests without manually managing a listening port.
