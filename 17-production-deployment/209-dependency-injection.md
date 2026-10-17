# Lesson 209 — Dependency Injection Concepts

## 1. What is a dependency?
A service needs other components to work: database repository, email provider, logger or clock. **Dependency injection (DI)** means providing those dependencies from outside instead of constructing them inside the service.

## 2. Before DI: tightly coupled code
```js
import { User } from "./user.model.js";
async function registerUser(email) {
  return User.create({ email }); // tied directly to Mongoose
}
```
A unit test now needs a database or complicated module mocking.

## 3. After DI: dependencies as parameters
```js
export function makeUserService({ users, mailer }) {
  return {
    async register(email) {
      if (await users.findByEmail(email)) throw new Error("Email exists");
      const user = await users.create({ email });
      await mailer.sendWelcome(user.email);
      return user;
    },
  };
}
```

**Production wiring (composition root):**
```js
const userService = makeUserService({
  users: mongooseUserRepository,
  mailer: brevoMailer,
});
```

**Unit test:**
```js
import test from "node:test";
import assert from "node:assert/strict";

test("register calls email provider", async () => {
  let sentTo;
  const service = makeUserService({
    users: {
      findByEmail: async () => null,
      create: async ({ email }) => ({ id: "u1", email }),
    },
    mailer: { sendWelcome: async (email) => { sentTo = email; } },
  });
  const user = await service.register("a@example.com");
  assert.equal(user.id, "u1");
  assert.equal(sentTo, "a@example.com");
});
```
No real MongoDB or email provider required.

## 4. Constructor / function / property injection
- **Constructor injection:** pass dependencies to a `new Service(deps)` constructor.
- **Function injection:** pass to factory `makeService(deps)`.
- **Method injection:** pass dependency into one call.
For most small Express apps, a factory is simple and effective.

## 5. Dependency inversion
High-level business rules should depend on a **contract** (`users.findByEmail`), not a specific Mongoose model. JavaScript does not require an interface syntax; document the shape and test it.

DI is **not** the same as “install a DI framework.” Containers can help large dependency graphs but are optional.

## 6. Error handling, transactions, lifetime
Inject dependencies with the appropriate lifetime. A repository should usually share an initialized DB connection, not call `mongoose.connect()` per method. If a service spans transactions, decide how to pass a transaction/session boundary safely.

In the registration example, a failed welcome email after saving a user may cause a partial success. For reliability, store an outbox event/queue a job rather than treating remote email as part of a database transaction.

## 7. Common mistakes
Passing the entire Express `app` as dependency; global mutable singleton test state; creating fresh DB connections in services; hiding dependencies behind magic imports; adding a DI container to two trivial functions.

## Practice
Refactor an order service to accept `orders`, `inventory`, and `clock`. Test it with fake implementations and no database.

**Interview answer:** Dependency injection supplies collaborators to business code from outside, reducing coupling and enabling lightweight tests. Factory functions are often enough; use containers only when complexity justifies them.
