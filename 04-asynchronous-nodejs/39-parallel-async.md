# Lesson 39 — Parallel Async Operations

## Why this lesson matters

A major Node.js performance mistake is awaiting independent I/O operations one by one.

If tasks do not depend on each other, they can often run concurrently.

This reduces total latency.

---

## 1. Parallel vs concurrent in Node.js

In everyday backend discussions, developers often say "parallel" when they mean "start these async I/O operations together."

Strictly:

- concurrency = tasks overlap in time
- parallelism = tasks literally execute simultaneously

For I/O in Node.js, the practical pattern is usually **concurrent async execution**.

---

## 2. Sequential version

```js
const profile = await getProfile();

const notifications =
  await getNotifications();

const settings = await getSettings();
```

If each takes 500 ms:

```text
profile         500 ms
notifications +500 ms
settings      +500 ms
---------------------
total         ~1500 ms
```

---

## 3. Concurrent version

```js
const [
  profile,
  notifications,
  settings,
] = await Promise.all([
  getProfile(),
  getNotifications(),
  getSettings(),
]);
```

All tasks start together.

Approximate total:

```text
max(500, 500, 500)
≈ 500 ms
```

---

## 4. Timeline mental model

Sequential:

```text
Task A: [------]
Task B:        [------]
Task C:               [------]
```

Concurrent:

```text
Task A: [------]
Task B: [------]
Task C: [------]
```

This is one of the easiest ways to explain the benefit in interviews.

---

## 5. Start promises first, await later

Another pattern:

```js
const profilePromise = getProfile();

const notificationsPromise =
  getNotifications();

const profile = await profilePromise;

const notifications =
  await notificationsPromise;
```

Both operations begin before the first await.

This is useful when:
- you need different error handling
- you want to do sync work between starts and awaits
- you need more control over orchestration

---

## 6. Real backend example

Dashboard endpoint:

```js
async function getDashboard(userId) {
  const [
    profile,
    orders,
    notifications,
    recommendations,
  ] = await Promise.all([
    getProfile(userId),
    getOrders(userId),
    getNotifications(userId),
    getRecommendations(userId),
  ]);

  return {
    profile,
    orders,
    notifications,
    recommendations,
  };
}
```

This is a classic production optimization.

---

## 7. Not all tasks should run concurrently

Bad:

```js
await Promise.all([
  createUser(),
  createProfile(user.id),
]);
```

Profile depends on user.

Concurrency is correct only for independent tasks.

---

## 8. Concurrency explosion

This can be dangerous:

```js
await Promise.all(
  100_000_users.map(
    (user) => sendEmail(user)
  )
);
```

This may create huge simultaneous load.

Possible problems:

- memory pressure
- database connection exhaustion
- API rate limit violations
- socket exhaustion
- CPU pressure
- provider throttling

---

## 9. Bounded concurrency

In production, you often want:

```text
100,000 jobs
     |
     v
process 10 at a time
```

This is called bounded concurrency.

Possible approaches:
- queue systems
- worker pools
- concurrency-limiting libraries
- batching

Conceptual implementation:

```text
batch 1 -> 10 tasks
batch 2 -> 10 tasks
batch 3 -> 10 tasks
```

---

## 10. Batch processing example

```js
for (let i = 0; i < users.length; i += 10) {
  const batch = users.slice(i, i + 10);

  await Promise.all(
    batch.map((user) =>
      sendEmail(user)
    )
  );
}
```

This limits concurrency to 10 per batch.

It is simple, though queues may be better for large workloads.

---

## 11. External service limits

Imagine an API allows:

```text
10 requests/second
```

Starting 5,000 requests simultaneously is incorrect even if technically possible.

Concurrency design must respect:
- provider rate limits
- database pool size
- infrastructure limits
- business rules

---

## 12. CPU-heavy operations

Promise concurrency does not automatically give CPU parallelism.

Example:

```js
await Promise.all([
  heavyCpuTask(),
  heavyCpuTask(),
]);
```

If both functions execute heavy synchronous JavaScript, they still block the same main thread.

For real CPU parallelism consider:
- Worker Threads
- child processes
- separate services

---

## 13. Parallel independent queries

Example:

```js
const [
  user,
  permissions,
] = await Promise.all([
  userRepository.findById(id),
  permissionRepository.findByUserId(id),
]);
```

This is useful only if the second query does not require data from the first.

---

## 14. Error behavior matters

With:

```js
await Promise.all([
  taskA(),
  taskB(),
  taskC(),
]);
```

one rejection causes `Promise.all` to reject.

But the other tasks may still continue.

This matters in side-effecting operations.

---

## 15. Common mistakes

### Mistake 1 — Parallelizing dependent tasks

Incorrect logical flow.

### Mistake 2 — Unlimited Promise.all

Can overwhelm resources.

### Mistake 3 — Assuming concurrent promises use separate CPU threads

They usually do not for JavaScript execution.

### Mistake 4 — Using Promise.all for side effects without understanding failure semantics

Partial side effects may still happen.

---

## 16. Interview questions

### When should async operations run concurrently?

When operations are independent and can safely start at the same time.

### Why is Promise.all faster than sequential awaits?

Because independent operations overlap their waiting time instead of waiting one after another.

### Can Promise.all overwhelm a system?

Yes. Large unbounded concurrency can exhaust connections, memory, sockets, or external rate limits.

### Does Promise.all make CPU work parallel?

No. It does not automatically move JavaScript computation to other threads.

---

## 17. Strong interview answer

> Independent I/O operations should often be started concurrently to reduce latency. Promise.all is a common way to do that. For example, if three independent database or API calls each take 500 milliseconds, sequential awaits may take around 1.5 seconds, while concurrent execution may complete in roughly the duration of the slowest call. However, concurrency should be bounded in large workloads because unlimited Promise.all can exhaust resources or violate rate limits.

---

## Interview-Ready Summary

```text
Concurrent async work
   |
   +--> independent tasks
   +--> lower latency
   +--> Promise.all
   +--> start promises early
   +--> bounded concurrency for large workloads

Avoid:
   dependent tasks in parallel
   huge unbounded Promise.all
   assuming CPU parallelism
```

## Practice Task

Build a dashboard service where:
- profile
- orders
- notifications
- recommendations

all load concurrently.

Then modify it so only 5 recommendation calls run at a time.
