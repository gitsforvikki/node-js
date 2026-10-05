# Lesson 36 — async / await

## Why async/await matters

Async/await is the most common way to write asynchronous application logic in modern Node.js.

It does not replace Promises.

It is syntax built on top of Promises.

That is one of the most important interview statements.

```text
async/await
    |
    v
Promise-based abstraction
```

---

## 1. async functions always return a Promise

```js
async function getValue() {
  return 42;
}
```

This function actually returns a Promise fulfilled with `42`.

```js
const result = getValue();

console.log(result);
```

`result` is a Promise.

Equivalent mental model:

```js
function getValue() {
  return Promise.resolve(42);
}
```

---

## 2. await waits for a Promise

```js
async function run() {
  const result = await Promise.resolve(42);

  console.log(result);
}

run();
```

Output:

```text
42
```

---

## 3. await does NOT block the whole Node.js process

This is critical.

```js
async function getUser() {
  const user = await db.users.findOne();

  return user;
}
```

While waiting for the database:

```text
getUser
   |
   | paused
   |
   +---------------------+
                         |
other requests execute  |
                         |
          DB responds
                         |
                         v
               getUser resumes
```

The current async function pauses.

The event loop remains free to process other work.

---

## 4. What happens conceptually at await?

Consider:

```js
async function example() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("1");

example();

console.log("2");
```

Output:

```text
1
A
2
B
```

Why?

Everything before `await` runs synchronously.

Continuation after `await` is scheduled asynchronously after the awaited value settles.

---

## 5. await with non-Promise values

```js
async function run() {
  const value = await 10;

  console.log(value);
}
```

This works.

Conceptually:

```js
await Promise.resolve(10);
```

---

## 6. Error handling with try/catch

```js
async function getUser(id) {
  try {
    const user = await db.users.findById(id);

    return user;
  } catch (error) {
    console.error(error);

    throw error;
  }
}
```

Rejected promises behave like thrown errors at the await point.

This gives asynchronous code a synchronous-looking error structure.

---

## 7. Sequential awaits

```js
const user = await getUser();

const orders = await getOrders(user.id);
```

This is correct because orders depend on user.

---

## 8. Accidental sequential execution

Consider:

```js
const profile = await getProfile();
const notifications = await getNotifications();
const balance = await getBalance();
```

If independent, this unnecessarily waits for each operation.

Better:

```js
const [
  profile,
  notifications,
  balance,
] = await Promise.all([
  getProfile(),
  getNotifications(),
  getBalance(),
]);
```

This is one of the most important performance patterns in backend Node.js.

---

## 9. Start first, await later

Another useful pattern:

```js
const profilePromise = getProfile();
const notificationPromise = getNotifications();

const profile = await profilePromise;
const notifications = await notificationPromise;
```

Both operations start before the first await.

This can be useful when logic needs more control than a single `Promise.all`.

---

## 10. Loops and async/await

### Sequential loop

```js
for (const id of userIds) {
  const user = await getUser(id);

  console.log(user);
}
```

This waits one by one.

This may be correct if:
- order matters
- rate limits exist
- each operation depends on the previous one

### Parallel

```js
const users = await Promise.all(
  userIds.map((id) => getUser(id))
);
```

This starts all operations together.

Be careful with huge arrays.

---

## 11. forEach + async trap

This is a classic interview question.

Bad:

```js
userIds.forEach(async (id) => {
  await deleteUser(id);
});

console.log("done");
```

`forEach` does not wait for returned promises.

`done` may log before deletions finish.

Better sequential:

```js
for (const id of userIds) {
  await deleteUser(id);
}
```

Better parallel:

```js
await Promise.all(
  userIds.map((id) => deleteUser(id))
);
```

---

## 12. Async map returns promises

```js
const result = users.map(async (user) => {
  return getProfile(user.id);
});
```

`result` is:

```text
Promise[]
```

To resolve:

```js
const profiles = await Promise.all(result);
```

---

## 13. Top-level await

In ES Modules, top-level await can be used:

```js
const config = await loadConfig();
```

Use carefully because module initialization can block dependent module loading until completion.

---

## 14. Async does not mean errors disappear

Bad:

```js
async function run() {
  await doSomethingRisky();
}

run();
```

If nobody handles rejection, you can create an unhandled rejection.

Better:

```js
run().catch((error) => {
  console.error(error);
});
```

or handle inside appropriate application boundaries.

---

## 15. Do not wrap async functions in new Promise

Bad:

```js
function getUser() {
  return new Promise(async (resolve, reject) => {
    try {
      const user = await db.users.findOne();
      resolve(user);
    } catch (error) {
      reject(error);
    }
  });
}
```

Better:

```js
async function getUser() {
  return db.users.findOne();
}
```

Using an async Promise executor is usually an unnecessary antipattern.

---

## 16. Real API example

```js
async function getDashboard(req, res, next) {
  try {
    const [
      profile,
      orders,
      notifications,
    ] = await Promise.all([
      profileService.getByUserId(req.user.id),
      orderService.getByUserId(req.user.id),
      notificationService.getByUserId(req.user.id),
    ]);

    res.json({
      profile,
      orders,
      notifications,
    });
  } catch (error) {
    next(error);
  }
}
```

This is clean, readable and concurrent.

---

## 17. Common mistakes

### Mistake 1 — Thinking async functions run on another thread

They do not automatically.

### Mistake 2 — Sequentially awaiting independent work

Creates latency.

### Mistake 3 — Using await inside forEach

Usually incorrect when you expect the outer flow to wait.

### Mistake 4 — Forgetting that async functions return Promises

Important for testing and error handling.

### Mistake 5 — Creating new Promise around async code unnecessarily

Avoid redundant wrapping.

---

## 18. Interview questions

### What does async do?

It makes a function return a Promise and allows use of `await` inside it.

### What does await do?

It pauses the current async function until the awaited promise settles, then resumes the function.

### Does await block Node.js?

No. It pauses only the current async function.

### Can await be used outside async?

Top-level await is supported in ES Modules, otherwise await is normally used inside async functions.

### Why is forEach with async problematic?

Because `forEach` does not wait for promises returned by its callback.

---

## 19. Strong interview answer

> Async/await is syntax built on top of Promises. An async function always returns a Promise. Await pauses only the current async function until the Promise settles, while the Node.js event loop can continue handling other work. Sequential awaits are appropriate for dependent operations, but independent operations should often be started concurrently using Promise.all or a similar pattern. Async/await improves readability, but it does not automatically create threads or make CPU-heavy work non-blocking.

---

## Interview-Ready Summary

```text
async function
   -> always returns Promise

await
   -> waits for settled value
   -> pauses current async function
   -> does not block Node.js

Dependent operations
   -> sequential await

Independent operations
   -> Promise.all

Avoid
   -> async forEach trap
   -> unnecessary new Promise(async ...)
```

## Practice Task

Build:

```text
getDashboard(userId)
```

Requirements:
- profile, notifications and recommendations run concurrently
- orders start only after user is loaded
- handle errors with try/catch
