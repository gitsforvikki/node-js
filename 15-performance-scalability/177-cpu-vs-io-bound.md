# Lesson 177 — CPU-bound vs I/O-bound Work

## 1. Two types of waiting
**CPU-bound** work spends most of its time calculating. **I/O-bound** work spends much of its time waiting for a database, network, filesystem or other external operation.

| Task | Usually | Why |
| --- | --- | --- |
| Loop that processes 100 million numbers | CPU-bound | JavaScript calculates |
| `await User.findById(id)` | I/O-bound | Wait for MongoDB |
| `await fetch(url)` | I/O-bound | Wait for remote response |
| Image encoding | CPU-bound | Many computations (often native code) |
| Reading file asynchronously | I/O-bound | Wait for system work |
| Synchronous JSON processing of enormous object | CPU-bound | Main JS thread computes |

An operation can contain **both** kinds of work (DB request + JSON serialization).

## 2. What happens inside Node?
```text
Main JavaScript thread / event loop
        |
        +--> JS calculations (stay on main thread)
        +--> async network I/O (OS networking + callbacks)
        +--> certain async fs/crypto/dns/zlib tasks (libuv worker pool)
        +--> Worker Threads for explicitly offloaded JavaScript CPU work
```
**Important:** `async/await` does not automatically move synchronous calculations onto another thread. It waits for Promises without blocking **only when the underlying operation is actually asynchronous**.

## 3. CPU-heavy synchronous route
```js
app.get("/calculate", (req, res) => {
  let sum = 0;
  for (let i = 0; i < 200_000_000; i++) sum += i;
  res.json({ sum });
});
```
While this loop runs, the main event loop cannot run ordinary JavaScript callbacks for other requests. The exact duration depends on hardware and optimization.

## 4. I/O-bound route
```js
app.get("/user/:id", async (req, res) => {
  const user = await User.findById(req.params.id).select("name").lean();
  if (!user) return res.sendStatus(404);
  res.json(user);
});
```
When waiting for MongoDB, Node can handle other I/O completions and requests. This is **concurrency**, not necessarily parallel JavaScript execution.

## 5. Two concurrent network calls
```js
const [profile, orders] = await Promise.all([
  fetchProfileFromService(),
  fetchOrdersFromService(),
]);
```
If independent, both operations can make progress concurrently. `Promise.all()` does not create threads or automatically cancel the other call if one rejects.

## 6. How to fix CPU-bound bottlenecks
- Move expensive JavaScript computation to **Worker Threads** (Lesson 187).
- Use a separate job worker/process for long-running tasks.
- Break work into small chunks when responsiveness matters, while measuring total cost.
- Reduce algorithmic complexity and cache results where appropriate.

**Do not use `await Promise.resolve()` as a fix:** resolved microtasks can still starve I/O; yielding to the event loop or offloading is different.

## 7. libuv worker-pool caution
Async `fs`, some `crypto`, DNS lookup and compression operations can use libuv's worker pool. The default pool is limited; overloading it causes queueing. Raising `UV_THREADPOOL_SIZE` blindly may increase resource contention and is not a substitute for measuring.

## Practice
Predict whether concurrent HTTP requests remain responsive if one route loops for 10 seconds. Compare with a route that `await`s a 10-second timer.

**Interview answer:** CPU-bound tasks spend time computing and can block the JavaScript event loop; I/O-bound tasks spend time waiting and are well served by Node's asynchronous I/O. Choose worker threads/processes for heavy JS computation, not for ordinary database queries.
