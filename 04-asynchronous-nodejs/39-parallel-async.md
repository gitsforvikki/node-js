# Lesson 39 — Concurrent Async Operations (Promise.all, allSettled, race)

## What will you learn?

- Sequential vs concurrent asynchronous operations
- Why Promise concurrency is not necessarily CPU parallelism
- Complete definitions, calls and outputs for **Promise.all()**, **Promise.allSettled()** and **Promise.race()**
- What happens when one Promise fails
- When to use each method in backend applications
- Dependent vs independent tasks, bounded concurrency, and interview questions

> **Running examples:** Use Node.js 18+ (or a modern browser). Every numbered JavaScript example below is **standalone**: paste a whole code block into a separate file and run it with `node example.js`. The displayed timings are approximate. Examples deliberately use `setTimeout` to simulate I/O such as database queries or API requests.

---

## 1. What does concurrency mean in Node.js?

Imagine a dashboard has three **independent** requests:

1. Fetch user profile (1 second)
2. Fetch notifications (2 seconds)
3. Fetch settings (1 second)

If we wait for the first, then start the second, then start the third, we waste time waiting. For independent I/O, we can **start all three together**.

- **Sequential:** One task starts after the previous task finishes.
- **Concurrent:** Multiple tasks make progress over overlapping time periods.
- **Parallel:** Work literally executes at the same instant on different execution resources.

In everyday Node.js discussions, people often call concurrent I/O "parallel async". Technically `Promise.all()` coordinates concurrency; it **does not automatically create JavaScript worker threads**.

## 2. Common helper: How do these async functions work?

This standalone example defines all functions. Here `setTimeout` simulates asynchronous work; a real application would use a database driver, `fetch()`, etc.

```js
function getProfile() {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ name: "Vikash" }), 1000);
  });
}

function getNotifications() {
  return new Promise((resolve) => {
    setTimeout(() => resolve(["New message", "Interview update"]), 2000);
  });
}

function getSettings() {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ theme: "dark" }), 1000);
  });
}

async function main() {
  const profile = await getProfile();
  console.log("Profile:", profile);

  const notifications = await getNotifications();
  console.log("Notifications:", notifications);

  const settings = await getSettings();
  console.log("Settings:", settings);
}

main().catch(console.error);
```

**Output (sequential):**

```text
`1s: Profile: { name: 'Vikash' }
`3s: Notifications: [ 'New message', 'Interview update' ]
`4s: Settings: { theme: 'dark' }
```

The `1s` / `3s` / `4s` labels above show approximate elapsed time, not literal text logged by JavaScript.

**Why `4 seconds?** Every `await` waits before the **next function is called**:

```text
Time:     0s       1s       2s       3s       4s
Profile:  [======]
Notifs:            [===============]
Settings:                            [======]
```

---

## 3. Promise.all() — all successes required

### What is Promise.all?

`Promise.all(iterable)` accepts Promises (or ordinary values) and returns **one Promise**.

- It fulfills when **all input Promises fulfill**.
- The result is an **array in input order**, even if tasks finish in a different order.
- If **any Promise rejects**, the combined Promise rejects with that reason.
- It **does not cancel** the other operations when one rejects.
- With an empty array, it fulfills with `[]`.

**When to use:** You need **all results** to build a response, for example profile + orders + settings on a dashboard.

### Complete working success example

```js
function getProfile() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Profile fetched");
      resolve({ id: 1, name: "Vikash" });
    }, 1000);
  });
}

function getNotifications() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Notifications fetched");
      resolve(["New message"]);
    }, 2000);
  });
}

function getSettings() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Settings fetched");
      resolve({ theme: "dark" });
    }, 500);
  });
}

async function loadDashboard() {
  try {
    console.log("Start dashboard requests");

    const [profile, notifications, settings] = await Promise.all([
      getProfile(),
      getNotifications(),
      getSettings(),
    ]);

    console.log("Final result:", { profile, notifications, settings });
  } catch (error) {
    console.error("Dashboard error:", error.message);
  }
}

loadDashboard();
```

**Expected output (approximately 2 seconds total):**

```text
Start dashboard requests
Settings fetched
Profile fetched
Notifications fetched
Final result: {
  profile: { id: 1, name: 'Vikash' },
  notifications: [ 'New message' ],
  settings: { theme: 'dark' }
}
```

**Execution:** Calling each function immediately starts its timer. All three timers overlap. Settings finishes first (500ms), profile second (1s), notifications last (2s). `Promise.all` returns results in **the original array order**, not completion order.

```text
Start together:
Settings:      [=====]                 0.5s
Profile:       [==========]            1.0s
Notifications: [====================]  2.0s
                                      |
Promise.all returns after slowest task: `2s
```

### Promise.all() when one task fails

```js
function getUser() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("User completed");
      resolve({ name: "Vikash" });
    }, 1000);
  });
}

function getOrders() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      console.log("Orders failed");
      reject(new Error("Database unavailable"));
    }, 500);
  });
}

function getSettings() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Settings completed");
      resolve({ theme: "dark" });
    }, 1500);
  });
}

async function main() {
  try {
    const result = await Promise.all([
      getUser(),
      getOrders(),
      getSettings(),
    ]);
    console.log("All results:", result); // Will not run
  } catch (error) {
    console.log("Caught:", error.message);
  }
}

main();
```

**Expected output:**

```text
Orders failed
Caught: Database unavailable
User completed
Settings completed
```

**Important:** `Promise.all` rejects as soon as `getOrders()` rejects, but **getUser() and getSettings() continue running**. This matters if requests charge money, send emails or write to the database. `Promise.all()` is not a transaction and does not automatically cancel work.

---

## 4. Promise.allSettled() — collect every outcome

### What is Promise.allSettled?

`Promise.allSettled(iterable)` waits until **every input Promise settles** (fulfilled or rejected). It **fulfills** with an array of result objects:

- Successful result: `{ status: "fulfilled", value: ... }`
- Failed result: `{ status: "rejected", reason: ... }`

The array follows **input order**, not completion order.

**When to use:** Tasks are independent, and you need to know which succeeded and which failed. Examples: fetch data from several services, check multiple APIs, send independent notifications and report per-item status.

### Complete working example: 2 successes + 1 failure

```js
function getProfile() {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ name: "Vikash" }), 1000);
  });
}

function getOrders() {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error("Orders service down")), 500);
  });
}

function getNotifications() {
  return new Promise((resolve) => {
    setTimeout(() => resolve(["New alert"]), 1500);
  });
}

async function main() {
  const results = await Promise.allSettled([
    getProfile(),
    getOrders(),
    getNotifications(),
  ]);

  // Print predictable values instead of the full Error stack.
  results.forEach((result, index) => {
    if (result.status === "fulfilled") {
      console.log("Task", index + 1, "SUCCESS:", result.value);
    } else {
      console.log("Task", index + 1, "FAILED:", result.reason.message);
    }
  });

  console.log("All tasks finished");
}

main().catch(console.error);
```

**Expected output (after `1.5 seconds):**

```text
Task 1 SUCCESS: { name: 'Vikash' }
Task 2 FAILED: Orders service down
Task 3 SUCCESS: [ 'New alert' ]
All tasks finished
```

**Step-by-step:**

1. Three Promise-returning functions are called immediately.
2. Orders rejects after 0.5s, profile fulfills after 1s, notifications fulfills after 1.5s.
3. `allSettled` waits for **all three**.
4. It returns results in input order, which is why `Task 1` is printed first even though Task 2 failed earlier.
5. We inspect each object's `status`, then read `value` or `reason`.

### Difference from Promise.all()

With the **same input**, `Promise.all()` would reject because orders failed. `Promise.allSettled()` still gives the profile and notifications result, **plus information about the failure**.

Notice: `Promise.allSettled()` itself generally fulfills; a rejected input is represented in its results, not thrown into an outer `catch`.

---

## 5. Promise.race() — first SETTLED result wins

### What is Promise.race?

`Promise.race(iterable)` returns a Promise that settles when **the first input Promise settles**.

- If the first settled Promise **fulfills**, race fulfills.
- If the first settled Promise **rejects**, race rejects.
- It does **not** wait for the remaining Promises to settle.
- It does **not** cancel the losing operations.
- An empty `Promise.race([])` remains pending.

**When to use:** A timeout guard, or a situation where you care about whichever task completes **or fails** first.

### Complete example: fastest successful task

```js
function fetchServerA() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Server A finished");
      resolve("Data from server A");
    }, 1500);
  });
}

function fetchServerB() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Server B finished");
      resolve("Data from server B");
    }, 500);
  });
}

async function main() {
  const winner = await Promise.race([
    fetchServerA(),
    fetchServerB(),
  ]);

  console.log("Race winner:", winner);
}

main().catch(console.error);
```

**Expected output:**

```text
Server B finished
Race winner: Data from server B
Server A finished
```

**Why does Server A still print?** `race` chooses the first settled Promise; it does **not stop** the timer for Server A.

### Important: First rejection can win too

```js
function fastFailure() {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error("Fast API failed")), 300);
  });
}

function slowSuccess() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Slow API finished");
      resolve("Successful data");
    }, 1000);
  });
}

async function main() {
  try {
    const result = await Promise.race([
      fastFailure(),
      slowSuccess(),
    ]);
    console.log(result); // Will not run
  } catch (error) {
    console.log("Race failed:", error.message);
  }
}

main();
```

**Expected output:**

```text
Race failed: Fast API failed
Slow API finished
```

So **race does not mean first successful result**. It means **first settled result**, success **or** failure.

> Related method: `Promise.any()` returns the **first fulfilled** result, ignoring earlier rejections as long as another Promise succeeds. If **all** reject, it rejects with an `AggregateError`. Do not confuse `any` with `race`.

### Practical timeout with Promise.race

This example shows how a Promise can time out **from the caller's perspective**.

```js
function fetchReport() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Report request finished in background");
      resolve("Report ready");
    }, 2000);
  });
}

function timeoutAfter(ms) {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error("Request timed out")), ms);
  });
}

async function main() {
  try {
    const report = await Promise.race([
      fetchReport(),
      timeoutAfter(500),
    ]);
    console.log("Report:", report);
  } catch (error) {
    console.log("Error:", error.message);
  }
}

main();
```

**Expected output:**

```text
Error: Request timed out
Report request finished in background
```

**Production warning:** This timeout does **not cancel** the underlying request. For cancellation, use a supported cancellation API such as `AbortController` with `fetch()`, plus appropriate cleanup. A timeout alone is not cancellation.

---

## 6. Promise.all vs allSettled vs race — quick comparison

| Method | When does it finish? | If one rejects | Returned value |
| --- | --- | --- | --- |
| `Promise.all()` | All fulfill, or first rejection | Combined Promise rejects early | Array of fulfilled values |
| `Promise.allSettled()` | Every input settles | Still collects every outcome | Array of objects with `status` + `value` / `reason` |
| `Promise.race()` | First input settles | Rejects **if that input settles first** | First result or rejection |

**Remember:** None of these methods automatically cancel still-running operations.

### Choosing the right method

- **Dashboard requires all data:** `Promise.all()`.
- **Show partial results even if one service fails:** `Promise.allSettled()`.
- **Take first finished response or enforce a timeout:** `Promise.race()`.
- **Take first successful response:** `Promise.any()` (see section 7).

## 7. Bonus: Promise.any() — first SUCCESS wins

```js
function apiA() {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error("API A failed")), 300);
  });
}

function apiB() {
  return new Promise((resolve) => {
    setTimeout(() => resolve("API B data"), 800);
  });
}

function apiC() {
  return new Promise((resolve) => {
    setTimeout(() => resolve("API C data"), 1200);
  });
}

async function main() {
  try {
    const data = await Promise.any([apiA(), apiB(), apiC()]);
    console.log("First success:", data);
  } catch (error) {
    console.log("All promises failed:", error.errors);
  }
}

main();
```

**Output (after `800ms):**

```text
First success: API B data
```

If **all Promises reject**, `Promise.any()` rejects with `AggregateError`, whose `errors` property lists their rejection reasons. Like `race`, it does not cancel other operations.

---

## 8. Independent tasks vs dependent tasks

### Wrong: Starting a dependent task too early

```js
// WRONG: user is not defined yet.
// await Promise.all([createUser(), createProfile(user.id)]);
```

`createProfile` needs the new user ID. They are **dependent** and must not be started together.

### Working example: do the dependent step first, then independent steps

```js
function createUser() {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ id: 10, name: "Vikash" }), 500);
  });
}

function createProfile(userId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ userId, bio: "Developer" }), 500);
  });
}

function createSettings(userId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ userId, theme: "dark" }), 500);
  });
}

async function main() {
  const user = await createUser(); // Must finish first.

  // These two now have the user ID and are independent.
  const [profile, settings] = await Promise.all([
    createProfile(user.id),
    createSettings(user.id),
  ]);

  console.log({ user, profile, settings });
}

main().catch(console.error);
```

**Expected output (`1 second total):**

```text
{
  user: { id: 10, name: 'Vikash' },
  profile: { userId: 10, bio: 'Developer' },
  settings: { userId: 10, theme: 'dark' }
}
```

**Rule:** First identify dependencies, then run independent steps concurrently.

---

## 9. Beware: Unlimited concurrency

This starts all operations almost immediately:

```js
// Dangerous with very large input:
// await Promise.all(users.map((user) => sendEmail(user)));
```

For 100,000 users this can overwhelm the email provider, memory, database connections or socket limits.

### Complete runnable example: process 2 users at a time

```js
const users = ["A", "B", "C", "D", "E"];

function sendEmail(user) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Email sent to", user);
      resolve(user);
    }, 500);
  });
}

async function main() {
  const batchSize = 2;

  for (let i = 0; i < users.length; i += batchSize) {
    const batch = users.slice(i, i + batchSize);
    await Promise.all(batch.map((user) => sendEmail(user)));
    console.log("Batch completed");
  }

  console.log("All emails processed");
}

main().catch(console.error);
```

**Expected output:**

```text
Email sent to A
Email sent to B
Batch completed
Email sent to C
Email sent to D
Batch completed
Email sent to E
Batch completed
All emails processed
```

Each batch has at most **2 active sendEmail operations**. This is **batch-based bounded concurrency**, not a general-purpose rolling worker pool. Production queues/pools may be more efficient; also respect request-per-second rate limits, which are different from concurrency limits.

---

## 10. Does Promise.all make CPU operations parallel?

**No.** `Promise.all()` doesn't create extra JavaScript threads. Calling synchronous CPU-heavy functions still blocks the main thread, even if the values are passed into `Promise.all()`.

For genuinely parallel CPU work, look at Node.js **worker_threads**, child processes, or other execution resources. Concurrent I/O and CPU parallelism are different ideas.

### Common mistakes

1. Thinking `Promise.all` starts functions itself: calling `getProfile()` **creates/starts** its async work; `Promise.all` combines returned promises.
2. Assuming `Promise.all` returns results in completion order: it returns them in **input order**.
3. Assuming `Promise.all` cancels tasks on failure: it doesn't.
4. Assuming `Promise.allSettled` throws if one input rejects: it normally **fulfills with rejection details**.
5. Assuming `Promise.race` returns the first success: a fast rejection can win too.
6. Running dependent tasks together.
7. Starting huge unbounded workloads without a concurrency limit.

---

## 11. Interview questions with short answers

**Q1. Why is Promise.all faster than sequential awaits for independent I/O?**

Because independent operations start together and their waiting times overlap. Total time is often roughly the duration of the slowest one, rather than the sum.

**Q2. What happens if one Promise rejects in Promise.all?**

The combined Promise rejects. Other already-started tasks continue unless you cancel them separately.

**Q3. How is Promise.allSettled different?**

It waits for every outcome and returns per-Promise `fulfilled` or `rejected` objects. Useful when partial success is acceptable.

**Q4. Difference between Promise.race and Promise.any?**

`race` chooses the first **settled** input; `any` chooses the first **fulfilled** input.

**Q5. Does Promise.all do parallel CPU processing?**

No. It coordinates Promise results and often concurrent I/O; it does not itself run synchronous JavaScript on several threads.

**Q6. How to prevent 100,000 requests running together?**

Use controlled batch sizes, a worker pool, a queue, and appropriate rate limits.

---

## 12. Practice tasks

1. Modify `Promise.all()` example so `getSettings()` rejects. Predict the console output.
2. Use `Promise.allSettled()` on 4 operations: 2 successes and 2 failures. Print only successful values.
3. In `Promise.race()`, make Server A reject faster than Server B succeeds. Predict the result.
4. Change the email batching example to process **3 users per batch**.
5. Create a dashboard where `getProfile`, `getOrders`, and `getNotifications` run concurrently, but `getPayment(orderId)` starts only after orders are returned.

**One-line summary:** `all` = need **every success**; `allSettled` = need **every outcome**; `race` = need **first settlement**; `any` = need **first success**.
