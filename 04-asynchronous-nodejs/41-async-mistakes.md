# Lesson 41 — Common Async Programming Mistakes

## Why this lesson matters

Many Node.js production bugs are not caused by syntax errors.

They come from async control-flow mistakes.

This lesson collects the most important ones that appear in:

- interviews
- backend APIs
- database code
- queues
- integrations
- real production systems

---

# 1. Forgetting await

Bad:

```js
const user = getUser();

console.log(user.name);
```

`user` is a Promise, not the resolved object.

Correct:

```js
const user = await getUser();
```

---

# 2. Forgetting to return a Promise

Bad:

```js
function getUser() {
  fetchUserFromDb();
}
```

Caller:

```js
await getUser();
```

This may resolve immediately because the internal Promise was not returned.

Correct:

```js
function getUser() {
  return fetchUserFromDb();
}
```

or:

```js
async function getUser() {
  return fetchUserFromDb();
}
```

---

# 3. Sequential awaits for independent work

Bad:

```js
const a = await getA();
const b = await getB();
const c = await getC();
```

If independent:

```js
const [a, b, c] = await Promise.all([
  getA(),
  getB(),
  getC(),
]);
```

This can drastically reduce latency.

---

# 4. Promise.all for dependent operations

Bad:

```js
await Promise.all([
  createUser(),
  createProfile(user.id),
]);
```

If profile depends on user, this is logically incorrect.

Use sequential execution.

---

# 5. async forEach

Classic trap:

```js
users.forEach(async (user) => {
  await sendEmail(user);
});

console.log("done");
```

`forEach` does not await returned Promises.

Use sequential:

```js
for (const user of users) {
  await sendEmail(user);
}
```

or concurrent:

```js
await Promise.all(
  users.map((user) =>
    sendEmail(user)
  )
);
```

---

# 6. async map without Promise.all

```js
const users = ids.map(async (id) => {
  return getUser(id);
});
```

Result:

```text
Promise[]
```

Correct:

```js
const users = await Promise.all(
  ids.map((id) => getUser(id))
);
```

---

# 7. Unhandled Promise rejection

Bad:

```js
async function run() {
  throw new Error("Failure");
}

run();
```

If the returned promise is ignored, rejection may be unhandled.

Better:

```js
run().catch((error) => {
  console.error(error);
});
```

---

# 8. Fire-and-forget without handling errors

Bad:

```js
sendAnalytics();
```

If it returns a Promise and rejects, nobody handles it.

Better:

```js
void sendAnalytics().catch((error) => {
  logger.error(error);
});
```

---

# 9. Swallowing errors

Bad:

```js
try {
  await processPayment();
} catch (error) {
  console.log(error);
}
```

If caller needs failure information, rethrow or translate.

```js
try {
  await processPayment();
} catch (error) {
  logger.error(error);

  throw new PaymentError(
    "Payment failed"
  );
}
```

---

# 10. new Promise(async ...)

Bad:

```js
return new Promise(
  async (resolve, reject) => {
    try {
      const user = await getUser();
      resolve(user);
    } catch (error) {
      reject(error);
    }
  }
);
```

Usually unnecessary.

Better:

```js
async function loadUser() {
  return getUser();
}
```

---

# 11. Blocking CPU work inside async function

Bad assumption:

```js
async function calculate() {
  for (let i = 0; i < 10_000_000_000; i++) {
    // CPU work
  }
}
```

The `async` keyword does not make CPU work non-blocking.

Use:
- Worker Threads
- background workers
- separate processes

---

# 12. Synchronous APIs in request handlers

Bad:

```js
app.get("/report", (req, res) => {
  const data =
    fs.readFileSync("large.csv");

  res.send(data);
});
```

This blocks the event loop.

Use async APIs or streams.

---

# 13. Unlimited concurrency

Bad:

```js
await Promise.all(
  users.map(sendEmail)
);
```

If users contains 1,000,000 items, this may overload the system.

Use bounded concurrency.

---

# 14. Assuming Promise.all cancels other work

Bad assumption:

```js
await Promise.all([
  updateOrder(),
  chargeCard(),
  sendEmail(),
]);
```

If `chargeCard` fails:
- updateOrder may already complete
- sendEmail may still continue

Promise.all is not a transaction.

---

# 15. Missing timeouts

Bad:

```js
const response = await fetch(externalUrl);
```

An external service may hang for too long.

Production integrations should consider:
- timeout
- AbortController
- retry strategy
- circuit breaker

---

# 16. Blind retries

Retrying every failure can make outages worse.

Bad:

```text
service failing
   |
   v
retry immediately
   |
   v
more load
   |
   v
service fails harder
```

Better strategies:
- exponential backoff
- jitter
- retry only transient failures
- retry limits

---

# 17. No idempotency for retried writes

Suppose payment creation times out.

You retry:

```text
request 1 -> payment created
response lost

retry
request 2 -> second payment created
```

This can cause duplicate side effects.

Use idempotency keys for critical retried writes.

---

# 18. Mixing callbacks and promises badly

Bad:

```js
function loadUser(callback) {
  getUser()
    .then((user) => {
      callback(null, user);
    })
    .catch((error) => {
      callback(error);
    });
}
```

Sometimes necessary for interoperability, but excessive mixing makes control flow harder.

Prefer one async abstraction per layer.

---

# 19. Forgetting cleanup

Bad:

```js
const connection =
  await pool.getConnection();

await doWork(connection);

connection.release();
```

If `doWork` throws, release never runs.

Better:

```js
const connection =
  await pool.getConnection();

try {
  await doWork(connection);
} finally {
  connection.release();
}
```

---

# 20. Catching too broadly

Bad:

```js
try {
  // 100 lines of unrelated logic
} catch {
  return null;
}
```

This hides bugs.

Catch errors where you can:
- recover
- translate
- add context
- clean up

---

# 21. Returning before async work finishes

Example:

```js
async function createUser() {
  saveUser();

  return {
    success: true,
  };
}
```

If `saveUser()` returns a Promise but is not awaited or returned, the function may report success before persistence finishes.

---

# 22. Forgetting async boundary errors

Background handlers need explicit error boundaries.

Example:

```js
queue.on("job", async (job) => {
  await processJob(job);
});
```

If framework/event system does not observe returned promises, failures may become unhandled.

Understand the contract of the callback API.

---

# 23. Race conditions

Async code can still create race conditions even though JavaScript execution is single-threaded.

Example:

```text
Request A reads stock = 1
Request B reads stock = 1

A buys item
B buys item

stock oversold
```

The problem happens across multiple async operations.

Solutions:
- atomic database operations
- transactions
- locks
- optimistic concurrency

Single-threaded JavaScript does not eliminate distributed/state races.

---

# 24. Common interview traps

### Does async make code multi-threaded?

No.

### Does await block the event loop?

No.

### Does Promise.all cancel other promises?

No.

### Does async forEach wait?

No.

### Can async code have race conditions?

Yes.

### Does sequential await mean transaction safety?

No.

---

# 25. Production checklist

Before shipping async code, ask:

```text
Are tasks dependent?
   |
   +--> yes -> sequential
   |
   +--> no -> concurrent?

Is concurrency bounded?

Are errors handled?

Are timeouts configured?

Are retries safe?

Are writes idempotent?

Is cleanup guaranteed?

Can race conditions occur?

Are synchronous APIs blocking the event loop?
```

---

# 26. Strong interview answer

> The most common async mistakes in Node.js include forgetting to await or return promises, using async callbacks with forEach, sequentially awaiting independent operations, using Promise.all for dependent or huge workloads, leaving unhandled rejections, blocking the event loop with CPU-heavy or synchronous work, missing timeouts and cleanup, and assuming Promise.all behaves like a transaction. Good async code requires thinking about dependency, concurrency limits, failure propagation, cancellation, idempotency, and shared-state race conditions.

---

## Interview-Ready Summary

```text
Avoid:
  forgotten await
  missing return
  async forEach
  unnecessary sequential waits
  unsafe Promise.all
  unbounded concurrency
  unhandled rejections
  swallowed errors
  blocking CPU work
  sync I/O in routes
  missing timeout
  unsafe retries
  missing cleanup
  race conditions

Think:
  dependency
  concurrency
  errors
  timeout
  retries
  idempotency
  consistency
```

## Section 4 Final Revision Map

```text
Async Node.js
    |
    +--> sync vs async
    +--> callbacks
    +--> callback hell
    +--> promises
    +--> async/await
    +--> error handling
    +--> sequential execution
    +--> concurrent execution
    +--> promise combinators
    +--> production mistakes
```

If you understand this section deeply, you are ready for the next major topic:

> Node.js Event Loop and Runtime Internals

That section explains **why all of this async behavior works internally**.

## Practice Task

Audit one existing Node.js backend project and find:

1. unnecessary sequential awaits
2. async forEach usage
3. missing timeouts
4. unhandled promises
5. unsafe retries
6. sync filesystem calls
7. possible race conditions

Document what you found and how you would improve it.
