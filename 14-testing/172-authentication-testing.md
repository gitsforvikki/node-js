# Lesson 172 — Authentication and Authorization Testing

## 1. Why is auth testing important?

Authentication answers **who are you?** Authorization answers **what are you allowed to do?** A successful login test is not enough: you must also test expired tokens, invalid signatures, logout, and users attempting to access other people's resources.

```text
HTTP request
  -> token/session validation (401 if invalid)
  -> permission/ownership check (403 or 404)
  -> controller
  -> safe response
```

## 2. Real Express 5 bearer-auth example

Install:

```bash
npm install express jsonwebtoken
npm install -D supertest
```

**src/app.js**

```js
import express from "express";
import jwt from "jsonwebtoken";

export const app = express();
const secret = "TEST_ONLY_replace_in_production"; // Learning example only

function requireAuth(req, res, next) {
  const value = req.get("Authorization");
  const token = value?.startsWith("Bearer ") ? value.slice(7) : null;
  if (!token) return res.sendStatus(401);

  try {
    req.user = jwt.verify(token, secret, { algorithms: ["HS256"] });
    next();
  } catch {
    res.sendStatus(401);
  }
}

app.get("/me", requireAuth, (req, res) => {
  res.json({ id: req.user.sub });
});

app.get("/users/:id", requireAuth, (req, res) => {
  if (req.params.id !== req.user.sub) return res.sendStatus(403);
  res.json({ id: req.params.id });
});

export const testSecret = secret;
```

**Important:** Do **not** export an application signing secret from production code. This example exports a test-only constant to keep the first lesson runnable. A real app must load a secret through secure configuration and test with separate test credentials.

## 3. Supertest auth scenarios

**test/auth.test.js**

```js
import test from "node:test";
import assert from "node:assert/strict";
import request from "supertest";
import jwt from "jsonwebtoken";
import { app, testSecret } from "../src/app.js";

const validToken = jwt.sign({ sub: "u1" }, testSecret, {
  algorithm: "HS256", expiresIn: "10m",
});

test("rejects missing token with 401", async () => {
  const res = await request(app).get("/me");
  assert.equal(res.status, 401);
});

test("accepts valid token", async () => {
  const res = await request(app).get("/me")
    .set("Authorization", `Bearer ${validToken}`);
  assert.equal(res.status, 200);
  assert.equal(res.body.id, "u1");
});

test("rejects altered token", async () => {
  const res = await request(app).get("/me")
    .set("Authorization", `Bearer ${validToken.slice(0, -5)}oops`);
  assert.equal(res.status, 401);
});

test("prevents access to another user's data", async () => {
  const res = await request(app).get("/users/u2")
    .set("Authorization", `Bearer ${validToken}`);
  assert.equal(res.status, 403);
});
```

## 4. Expired and wrong-signature token tests

```js
test("rejects expired JWT", async () => {
  const expired = jwt.sign({ sub: "u1" }, testSecret, { expiresIn: -1 });
  const res = await request(app).get("/me")
    .set("Authorization", `Bearer ${expired}`);
  assert.equal(res.status, 401);
});

test("rejects JWT signed with another key", async () => {
  const wrong = jwt.sign({ sub: "u1" }, "different-key");
  const res = await request(app).get("/me")
    .set("Authorization", `Bearer ${wrong}`);
  assert.equal(res.status, 401);
});
```

In production, also verify issuer, audience, signing algorithm, and expected claim types.

## 5. Login tests

Test these with a test-only user database:
- Valid credentials → safe session/token, no password hash.
- Wrong password → 401, generic error.
- Unknown email → similar response (avoid account enumeration).
- Too many attempts → rate limiting where configured.
- Disabled user → rejected.
- Password hash verification uses bcrypt/Argon2, never plaintext comparison.

For login, use **real hashing** in a suitable integration test, not only a mocked `bcrypt.compare`.

## 6. Cookie session and CSRF tests

`request.agent(app)` can preserve cookies between calls.

```js
const agent = request.agent(app);
// After your real login endpoint sets the session cookie:
await agent.post("/login").send({ email, password });
await agent.get("/me").expect(200);
```

If the app uses cookie-based auth, test `HttpOnly`, `Secure`, `SameSite`, CSRF token validation, and logout/session invalidation.

## 7. Refresh tokens and logout

A secure auth test suite checks:
1. Access token expires, refresh issues a new access token.
2. Refresh token rotates if that is your architecture.
3. Old refresh token reuse is rejected or triggers reuse detection.
4. Logout invalidates server-side refresh sessions.
5. Revoked/expired tokens cannot create new sessions.

A stateless JWT access token may remain valid until expiry unless you implement explicit revocation; test the behavior your design actually promises.

## 8. Authorization tests are especially important

Test **BOLA/IDOR**: User A requests User B's order by guessing the order ID. The API must deny access even though User A is logged in.

Test roles: ordinary user cannot call admin endpoints; admin can perform only explicitly permitted actions. Validate ownership at the database/query boundary.

## 9. Common mistakes and practice

Mistakes: testing only login success, skipping negative cases, hardcoding real production secrets, confusing 401 and 403, using mock JWT decoding instead of verification, and forgetting logout/reset endpoints.

Practice: add invalid token, expired token, another user's order and admin role tests.

**Interview answer:** Auth testing verifies identity, token/session validity, expiry and revocation; authorization testing checks role and ownership for every protected action. Cover happy paths plus invalid credentials, tampering, expiry, logout and cross-user access.
