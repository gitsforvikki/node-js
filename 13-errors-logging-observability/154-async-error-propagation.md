# Lesson 154 — Async Error Propagation in Node.js and Express 5

## 1. What does propagation mean?

**Error propagation** means a failure travels from the place it happened to the code responsible for handling it.

```text
Database rejects Promise
   -> service await rejects
   -> controller await rejects
   -> Express 5 error middleware
   -> safe HTTP response
```

The key skill: know **who receives the error** when you use sync functions, Promises, `async/await`, callbacks, and event emitters.

## 2. Synchronous throw

```js
function parseAge(value) {
  if (typeof value !== "number") throw new Error("Age must be number");
  return value;
}

try {
  parseAge("abc");
} catch (err) {
  console.log(err.message); // Age must be number
}
```

`throw` exits the function; the nearest applicable `catch` receives it.

## 3. Promise rejection

```js
function getUser() {
  return Promise.reject(new Error("Database unavailable"));
}

getUser()
  .then((user) => console.log(user))
  .catch((err) => console.error("Caught:", err.message));
```

A Promise represents a future result. It can be **fulfilled** or **rejected**; `.catch()` handles the rejection.

## 4. async/await + try/catch

```js
async function loadUser() {
  const user = await getUser(); // rejection becomes a throw here
  return user;
}

try {
  await loadUser();
} catch (err) {
  console.error(err.message);
}
```

**Mental model:** inside an `async` function, `await rejectedPromise` behaves like throwing an error. The async function itself returns a **rejected Promise** unless handled.

## 5. Express 5 automatically propagates rejected handler Promises

```js
app.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id); // may reject
  if (!user) throw new AppError("User not found", 404, "NOT_FOUND");
  res.json(user);
});

app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);
  res.status(err instanceof AppError ? err.statusCode : 500)
    .json({ message: err instanceof AppError ? err.message : "Server error" });
});
```

No extra try/catch is required merely to forward Promise errors. In **Express 4**, rejected async handlers generally needed a wrapper or explicit `next(err)`.

## 6. Why `return` or `await` matters

```js
async function service() {
  // BAD: starts a Promise but does not wait for or return it
  saveToDatabase();
}

async function safeService() {
  // GOOD: rejection belongs to safeService's Promise
  return saveToDatabase();
  // or: await saveToDatabase();
}
```

If you don't return/await an async operation, its failure might become an **unhandled rejection** instead of reaching your request's error handler.

## 7. Errors in setTimeout do not reach surrounding try/catch

```js
// WRONG assumption: outer try/catch will NOT catch later callback throw
try {
  setTimeout(() => {
    throw new Error("Timer failed");
  }, 10);
} catch (err) {
  console.log("Never catches timer error");
}
```

Why? The outer `try` finishes **before** the callback runs in a later event-loop turn.

Instead catch inside callback or adapt to Promise:

```js
import { setTimeout as delay } from "node:timers/promises";

app.get("/demo", async (req, res) => {
  await delay(10);
  throw new Error("Caught by Express 5 via handler Promise");
});
```

## 8. Callback errors: error-first pattern

```js
import { readFile } from "node:fs";

readFile("./config.json", "utf8", (err, data) => {
  if (err) {
    console.error("File read failed:", err);
    return;
  }
  console.log(data);
});
```

The callback receives `(err, result)`. This is **different** from a Promise rejection; convert callback APIs to Promise APIs where appropriate (`node:fs/promises`, `promisify`).

## 9. EventEmitter and stream errors

Streams and EventEmitters may emit an **error event** rather than rejecting a Promise:

```js
import { createReadStream } from "node:fs";

const stream = createReadStream("./missing.txt");
stream.on("error", (err) => console.error("Stream error:", err.message));
```

An EventEmitter emitting `error` without an error listener can crash the process. For pipelines, prefer `pipeline` from `node:stream/promises`, which handles stream errors and rejects:

```js
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";

try {
  await pipeline(
    createReadStream("./source.txt"),
    createWriteStream("./copy.txt")
  );
} catch (err) {
  console.error("Copy failed:", err.message);
}
```

## 10. Detached/background work and request handlers

```js
app.post("/jobs", async (req, res) => {
  // BAD: unobserved, detached Promise
  runJob();
  res.sendStatus(202);
});
```

Use a real background job queue/worker for durable work. At minimum, explicitly handle the Promise failure:

```js
app.post("/jobs", (req, res) => {
  void runJob().catch((err) => {
    logger.error({ err }, "Background task failed");
  });
  res.sendStatus(202);
});
```

This catches errors but **does not make the task durable** (process restarts can lose it).

## 11. Double propagation mistake

```js
// BAD: next(err) plus rethrow can forward same error twice
app.get("/demo", async (req, res, next) => {
  try { await action(); }
  catch (err) {
    next(err);
    throw err;
  }
});
```

Choose one strategy: let the rejected handler Promise reach Express 5 automatically, **or** call `next(err)` and do not rethrow.

## 12. When do we still need try/catch?

Use it to:
- Recover from a known failure (for example, fallback cache).
- Translate known vendor errors into safe AppErrors.
- Roll back/compensate when needed.
- Add useful logging/context before rethrowing once.

Do not swallow errors and leave HTTP clients waiting.

## 13. Common mistakes and practice

Mistakes: missing `await`, detached Promises, assuming `setTimeout` throws reach outer catch, missing stream `error` listeners, calling `next(err)` and rethrowing.

Practice:
1. Predict output when an async function throws before its first `await`.
2. Create a controller that awaits a rejected DB Promise and confirm centralized middleware runs.
3. Convert an error-first callback to a Promise.
4. Explain why a timer callback error is not caught by outer try/catch.
5. Explain how `pipeline()` handles stream failures.

**Interview answer:** Sync throws, Promise rejections, callback errors, and EventEmitter errors have different propagation mechanisms. Express 5 catches rejected Promises from returned route/middleware handlers, but detached tasks and later callback throws need their own handling.
