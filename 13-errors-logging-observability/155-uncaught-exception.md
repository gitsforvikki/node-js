# Lesson 155 — uncaughtException

## 1. What does uncaughtException mean?

An **uncaught exception** is a JavaScript error that reaches the Node.js event loop without being handled. It is a **process-level failure**, not an ordinary Express request error.

Think of two situations:
- A user asks for a product that doesn't exist: respond with **404**, keep server running.
- A serious programming bug escapes all handlers: the **Node.js process may no longer be reliable**.

## 2. Example: a normal caught error

```js
try {
  JSON.parse("{invalid json}");
} catch (error) {
  console.log("Handled:", error.message);
}
console.log("Application continues");
```

Here, the error is caught. `uncaughtException` does **not** fire.

## 3. Example: an uncaught error

Save this in `crash.js` and run `node crash.js`:

```js
console.log("1. Application started");

setTimeout(() => {
  console.log("2. Timer callback running");
  throw new Error("Unexpected programmer bug");
}, 100);

console.log("3. Timer scheduled");
```

Typical output:

```text
1. Application started
3. Timer scheduled
2. Timer callback running
Error: Unexpected programmer bug
...
```

The callback runs later. The throw is not caught by any surrounding handler, so Node prints an error and normally exits with failure.

## 4. Why outer try/catch does not work for later callbacks

```js
try {
  setTimeout(() => {
    throw new Error("Boom");
  }, 100);
} catch (error) {
  console.log("This does NOT catch the timer error");
}
```

The outer `try` finishes **before** the timer executes. Catch errors inside the callback, or use Promise-based APIs and `await` when appropriate.

## 5. Listen to the event — for last-resort cleanup

```js
process.on("uncaughtException", (error, origin) => {
  console.error("Fatal uncaught exception:", error);
  console.error("Origin:", origin);
  process.exitCode = 1;
  // Begin a bounded, best-effort shutdown; then terminate.
});
```

**Important:** Installing this listener changes Node's default automatic-exit behavior. Merely setting `process.exitCode` is **not sufficient** if open servers or timers keep the event loop alive. A production fatal handler should stop accepting work, perform **bounded** cleanup, then terminate; use a process manager to restart the service.

Do not simply log and continue serving requests after an unexpected uncaught exception: application state may be inconsistent.

## 6. Crash diagnostics without overriding exit behavior

`uncaughtExceptionMonitor` lets you observe the error without changing Node's normal uncaught-exception exit behavior:

```js
process.on("uncaughtExceptionMonitor", (error, origin) => {
  console.error("Fatal error observed:", error.message, origin);
});
```

This is often preferable for diagnostic logging when you rely on Node's default crash and a supervisor restart.

## 7. Express 5 vs process-level failures

```js
app.get("/users", async (req, res) => {
  throw new Error("Query failed");
});
```

Express 5 forwards a rejection from the returned async handler to your **error middleware**, so this normally does not become an uncaught exception.

But:

```js
app.get("/unsafe", (req, res) => {
  setTimeout(() => {
    throw new Error("Detached timer bug");
  }, 100);
  res.sendStatus(202);
});
```

The later callback is **outside Express's handler Promise**. Handle the callback or move work into a managed background job.

## 8. Production recovery strategy

```text
Unexpected uncaught exception
  -> record safe diagnostic information
  -> mark instance unhealthy / stop accepting traffic
  -> attempt short bounded cleanup
  -> exit with non-zero status
  -> PM2/systemd/container orchestrator restarts instance
```

A supervisor restarts a crashed service; it does not fix the underlying bug. Investigate the stack trace and add a regression test.

## 9. Common mistakes

- Using `uncaughtException` as your normal API error handler.
- Calling `process.exit(0)` after a fatal error (falsely signals success).
- Keeping the server running after an unknown fatal failure.
- Assuming an outer try/catch captures errors thrown later by timers.
- Printing secrets or complete user requests in crash logs.
- Using `process.exit()` immediately when a bounded shutdown could flush critical logs safely.

## 10. Practice and interview

1. Run `crash.js` and explain output order.
2. Add `uncaughtExceptionMonitor`; confirm Node still exits on an uncaught exception.
3. Explain why Express 5 error middleware does not handle a detached timer throw.
4. Explain why a process manager is needed after fatal failures.

**Interview answer:** `uncaughtException` occurs when an exception escapes all handlers. It is a last-resort process event, not a normal request error. Log safely, terminate after bounded cleanup if handling it, and let a supervisor restart the service.
