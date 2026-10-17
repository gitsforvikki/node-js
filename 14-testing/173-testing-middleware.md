# Lesson 173 — Testing Express Middleware

## 1. What is middleware testing?

Express middleware runs between request and response. It may check authentication, validate input, rate-limit a client, attach a request ID, or handle errors.

Middleware tests verify **what it changes** and **whether it calls `next()` or sends a response**.

## 2. A simple middleware

**src/middleware/requireApiKey.js**
```js
export function requireApiKey(req, res, next) {
  const key = req.get("x-api-key");
  if (key !== process.env.API_KEY) {
    return res.status(401).json({ message: "Invalid API key" });
  }
  next();
}
```

For real products, static API keys need secure storage, careful rotation, transport security and rate limits; this is a minimal teaching example.

## 3. Integration-style middleware test with Supertest

```js
import express from "express";
import request from "supertest";
import test from "node:test";
import assert from "node:assert/strict";
import { requireApiKey } from "../src/middleware/requireApiKey.js";

const app = express();
app.get("/private", requireApiKey, (req, res) => res.json({ ok: true }));

test("blocks missing key", async () => {
  const res = await request(app).get("/private");
  assert.equal(res.status, 401);
});

test("allows configured key", async () => {
  const original = process.env.API_KEY;
  process.env.API_KEY = "test-only-api-key";
  try {
    const res = await request(app).get("/private")
      .set("x-api-key", "test-only-api-key");
    assert.equal(res.status, 200);
    assert.deepEqual(res.body, { ok: true });
  } finally {
    if (original === undefined) delete process.env.API_KEY;
    else process.env.API_KEY = original;
  }
});
```

**Note:** Tests modifying `process.env` should not run concurrently within the same process or share state unpredictably. A dependency-injected key provider is better for larger suites.

## 4. Unit test middleware using fake req/res/next

```js
import { test } from "node:test";
import assert from "node:assert/strict";

test("calls next when key matches", () => {
  const original = process.env.API_KEY;
  process.env.API_KEY = "abc";
  try {
    let nextCalled = false;
    const req = { get: () => "abc" };
    const res = {};
    requireApiKey(req, res, () => { nextCalled = true; });
    assert.equal(nextCalled, true);
  } finally {
    if (original === undefined) delete process.env.API_KEY;
    else process.env.API_KEY = original;
  }
});
```

For realistic HTTP response and headers behavior, **prefer Supertest integration tests** over attempting to mock every Express `res` method.

## 5. Test validation middleware

```js
import { z } from "zod";

const schema = z.strictObject({ name: z.string().min(2) });
const validate = (req, res, next) => {
  const result = schema.safeParse(req.body);
  if (!result.success) return res.sendStatus(400);
  res.locals.validatedBody = result.data;
  next();
};
```

Assert both branches:
- `{name:"Vikash"}` → route executes.
- `{name:""}` → 400 and route not executed.

## 6. Error middleware

An Express **error** middleware has **four arguments**: `(err, req, res, next)`.

```js
app.get("/broken", async () => {
  throw new Error("DB unavailable");
});

app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);
  res.status(500).json({ message: "Internal server error" });
});
```

Supertest should see a safe 500 body, not the private stack trace. In Express 5, rejected Promises from returned async handlers reach the error middleware automatically.

## 7. Order matters

```text
Request ID -> parsing -> auth -> validation -> controller
                                               |
                                               v
                                         error middleware
```

Register middleware before the routes it should affect, and error middleware **after** routes. Test that unauthorized requests do not reach protected business logic.

## 8. Rate limiter middleware testing

Use a tiny test-only limit and isolated rate-limiter store per test suite. Send repeated requests and assert HTTP 429. Avoid depending on real sleep periods; inject or reset time/state where your limiter supports it.

## 9. Common mistakes

- Forgetting to call `next()` when allowed.
- Calling `next()` **after** sending a response, causing double responses.
- Not testing blocked paths.
- Mounting error middleware before routes.
- Modifying shared env or mock state while tests run concurrently.
- Treating `next(err)` as an ordinary success continuation.

## 10. Practice

1. Test a `requireAdmin` middleware returning 403 for a normal user.
2. Test validation middleware with good and bad JSON.
3. Test a global error handler.
4. Verify request-ID middleware adds the header.

**Interview answer:** Middleware tests verify request transformation, access checks, error forwarding, and middleware ordering. Use small unit tests for pure logic and Supertest for realistic Express behavior.
