# Lesson 156 — unhandledRejection

## 1. What is an unhandled Promise rejection?

A Promise can **resolve** or **reject**. If it rejects and no rejection handler is attached within the relevant event-loop turn, Node can emit `unhandledRejection`.

Example: You call a database operation but forget to `await`, `return`, or `.catch()` it.

## 2. A simple example

```js
function getUser() {
  return Promise.reject(new Error("Database unavailable"));
}

getUser(); // Promise is rejected; nobody handles it
```

On modern Node.js defaults, an unhandled rejection can be raised as an uncaught exception and terminate the process. Behavior depends on Node version and `--unhandled-rejections` mode; do not rely on the process merely warning.

## 3. Three correct ways to handle rejection

### A. .catch()

```js
getUser().catch((error) => {
  console.error("Handled:", error.message);
});
```

### B. await + try/catch

```js
async function main() {
  try {
    const user = await getUser();
    console.log(user);
  } catch (error) {
    console.error("Handled:", error.message);
  }
}
await main();
```

### C. Return the Promise to the caller

```js
function loadUser() {
  return getUser(); // caller must handle rejection
}

loadUser().catch(console.error);
```

Returning a Promise transfers responsibility to the caller. It does **not** make the failure disappear.

## 4. The common missing-await bug

```js
async function createOrder() {
  saveOrderToDatabase(); // BAD: detached Promise
  return { ok: true };
}
```

If `saveOrderToDatabase()` rejects, the `createOrder()` Promise may still resolve successfully. The failure has become detached.

Fix:

```js
async function createOrder() {
  await saveOrderToDatabase();
  return { ok: true };
}
```

Now the rejection propagates to the caller.

## 5. Express 5: handled vs detached Promise

```js
// GOOD: Express 5 observes returned async handler Promise
app.post("/orders", async (req, res) => {
  const order = await Order.create(req.body);
  res.status(201).json(order);
});
```

```js
// BAD: sendEmail() rejection is not part of handler Promise
app.post("/orders", async (req, res) => {
  sendEmail();
  res.sendStatus(202);
});
```

For reliable background work, use a queue/worker. For a small non-durable task, attach `.catch()` and log failures, understanding that a process crash may lose the task.

## 6. The process event

```js
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
  // Treat as a serious process-level failure;
  // arrange controlled termination/restart where appropriate.
});
```

Installing this listener changes how default failure handling behaves. It is **not** permission to ignore rejected Promises. A robust app handles them at their source and has a fatal-error strategy for unexpected ones.

`reason` can be an Error, string, object, or other value because JavaScript permits `Promise.reject(anything)`.

## 7. Late handlers and rejectionHandled

A Promise can be initially unhandled and later receive a catch:

```js
const p = Promise.reject(new Error("Late catch"));

setTimeout(() => {
  p.catch(() => console.log("Handled later"));
}, 100);
```

Node may report `unhandledRejection` before the later handler and emit `rejectionHandled` afterward. With default fatal behavior the process may exit first, so **attach handlers immediately**.

## 8. Promise.all and parallel work

```js
try {
  const [user, orders] = await Promise.all([
    getUser(),
    getOrders(),
  ]);
} catch (error) {
  console.error("At least one operation failed:", error.message);
}
```

`Promise.all` rejects if one input rejects, but **other operations are not automatically cancelled**. For work with side effects, consider cancellation, idempotency, or compensation.

## 9. Startup failures

```js
async function start() {
  await connectDB();
  app.listen(3000);
}

start().catch((error) => {
  console.error("Startup failed:", error.message);
  process.exitCode = 1;
});
```

This works when there are no long-lived handles keeping the process alive. If a partial startup opened resources, close them and terminate explicitly. Do not start accepting traffic before critical dependencies are ready.

## 10. Common mistakes

- Forgetting `await` on database writes.
- Assuming `try/catch` catches a Promise that was never awaited.
- Adding a global listener instead of fixing the missing catch.
- Returning success before an essential operation completes.
- Assuming `Promise.all` cancels remaining tasks.
- Ignoring rejected background jobs.

## 11. Practice and interview

1. Predict what happens when `Promise.reject(new Error("fail"))` has no catch.
2. Rewrite a detached DB call using `await`.
3. Compare `Promise.all` and sequential `await` for failure behavior.
4. Explain why `unhandledRejection` is different from a normal 404 error.

**Interview answer:** `unhandledRejection` is emitted when a rejected Promise lacks a timely rejection handler. Fix it by awaiting, returning, or catching Promises. Express 5 handles returned async route rejections, but detached Promises still need explicit handling.
