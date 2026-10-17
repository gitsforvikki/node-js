# Lesson 169 — Supertest

## 1. What is Supertest?
**Supertest** sends HTTP-like requests to a Node/Express application and checks the response. It is useful for automated **API testing** without opening and managing a permanent listening server yourself.

## 2. Setup
```bash
npm install express
npm install -D supertest
```

Use ESM with `"type": "module"` and `node --test`.

**src/app.js**
```js
import express from "express";

export const app = express();
app.use(express.json());

const products = [{ id: "p1", name: "Mouse" }];

app.get("/products", (req, res) => res.json(products));
app.post("/products", (req, res) => {
  if (!req.body.name) return res.status(400).json({ message: "Name required" });
  res.status(201).json({ id: "p2", name: req.body.name });
});
```

**src/server.js**
```js
import { app } from "./app.js";
app.listen(3000, () => console.log("Server listening"));
```

## 3. First Supertest tests
**test/products.test.js**
```js
import test from "node:test";
import assert from "node:assert/strict";
import request from "supertest";
import { app } from "../src/app.js";

test("GET /products responds 200", async () => {
  const res = await request(app).get("/products");
  assert.equal(res.status, 200);
  assert.equal(res.body[0].name, "Mouse");
});

test("POST /products creates a product", async () => {
  const res = await request(app).post("/products")
    .send({ name: "Keyboard" })
    .expect("Content-Type", /json/)
    .expect(201);
  assert.equal(res.body.name, "Keyboard");
});

test("POST without name returns 400", async () => {
  const res = await request(app).post("/products").send({});
  assert.equal(res.status, 400);
});
```

Run `node --test`.

## 4. Understand the chain
```js
await request(app)      // select Express app
  .post("/products")    // HTTP method and path
  .set("Authorization", "Bearer example-token")
  .send({ name: "Pen" }) // JSON payload
  .expect(201);         // response assertion
```
The fake token above is for illustrating headers, not a valid login token.

## 5. Query, headers and cookies
```js
await request(app).get("/products").query({ page: 2 });
await request(app).get("/private").set("Authorization", `Bearer ${token}`);
```

For cookie sessions use `request.agent(app)` to keep cookies between requests:
```js
const agent = request.agent(app);
await agent.post("/login").send({ email, password });
await agent.get("/me").expect(200);
```
The second example requires a real /login and /me implementation. If your session uses CSRF protection, include the legitimate CSRF token in state-changing requests.

## 6. Important design: test isolation
Do not make API tests depend on hardcoded state shared across tests. This minimal example uses static data only; for database-backed tests, create fresh fixtures and clean each case.

## 7. Supertest vs Postman vs curl
Postman/curl: great for manual debugging. Supertest: repeatable automated assertions. Supertest runs against Express in the same test process; full E2E tests may exercise a deployed stack.

## 8. Common mistakes
Importing `server.js` and unintentionally starting a port; forgetting `await`; checking status only; reusing production DB; assuming Supertest automatically mocks MongoDB; ignoring error responses.

## 9. Practice
Add `GET /products/:id` (200/404) and test both. Test malformed POST body. Then test a protected endpoint with and without credentials.

**Interview answer:** Supertest makes HTTP assertions against Express apps, integrating naturally with Jest, Vitest or Node's test runner. Separate app configuration from `listen()` for reliable tests.
