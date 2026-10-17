# Lesson 181 — Memory Leaks in Node.js

## 1. What is a memory leak?
A **memory leak** happens when the application keeps memory alive longer than necessary, so memory use grows as work continues.

In garbage-collected JavaScript, leaks are often **unwanted references**, not a missing manual `free()` call.

## 2. Real-world example: unlimited cache
```js
const results = new Map();

app.get("/users/:id", async (req, res) => {
  if (!results.has(req.params.id)) {
    const user = await User.findById(req.params.id).lean();
    results.set(req.params.id, user);
  }
  res.json(results.get(req.params.id));
});
```
Each new ID remains in the Map forever. With many users, RAM keeps growing; deleted/updated users also become stale.

**Fix:** use bounded cache entries + TTL, or Redis with controlled expiry (Lessons 183–184). In this example, also handle missing users and validate IDs.

## 3. Event listener leak
```js
// BAD: adds one listener for every request and never removes it
app.get("/events", (req, res) => {
  eventBus.on("update", () => console.log("Update for request"));
  res.sendStatus(200);
});
```
Thousands of requests create thousands of listeners. Even after responses finish, the emitter retains them.

Better: use one process-wide listener, or attach per-request listeners with cleanup:
```js
app.get("/events", (req, res) => {
  const onUpdate = (data) => res.write(JSON.stringify(data) + "\n");
  eventBus.on("update", onUpdate);
  res.on("close", () => eventBus.off("update", onUpdate));
  res.setHeader("Content-Type", "text/plain");
  res.write("connected\n");
});
```
This illustrates cleanup, not a complete robust streaming/SSE implementation. Production long-lived responses also need auth, backpressure, timeouts and disconnect handling.

## 4. Timers and closures
A `setInterval` callback closes over referenced variables. If a timer remains active, those referenced objects can remain reachable.

```js
const timer = setInterval(doWork, 1000);
clearInterval(timer); // when the job is no longer needed
```
Other common leaks: unbounded queues, forgotten WebSocket connections, detached async operations, retaining req/res objects, and growing global arrays.

## 5. Detect a real leak
1. Generate a **representative repeatable load**.
2. Track `rss`, `heapUsed` and `external` over time.
3. Observe if **post-GC heap baseline** grows after repeated workloads.
4. Capture heap snapshots/profiles in a controlled environment.
5. Compare retained objects and **retaining paths**.
6. Fix reference ownership; repeat the same test.

A high RSS number by itself does not prove a leak: memory allocators, fragmentation and caches can grow then stabilize.

## 6. Quick memory monitor
```js
setInterval(() => {
  const { heapUsed, rss, external } = process.memoryUsage();
  console.log({
    heapMB: Math.round(heapUsed / 1024 / 1024),
    rssMB: Math.round(rss / 1024 / 1024),
    externalMB: Math.round(external / 1024 / 1024),
  });
}, 10_000).unref();
```

## 7. Heap snapshot cautions
```bash
node --inspect src/server.js
```
Connect Chrome DevTools only in a trusted environment and inspect **Memory → Heap snapshot**. Heap snapshots can pause the process, temporarily use considerable memory, and contain user data/secrets. Never expose the inspector port publicly.

## 8. Fixes
- Use LRU/TTL/max-size caches.
- Remove listeners and stop timers.
- Bound queues, uploads and concurrency.
- Drop references when no longer needed.
- Use `WeakMap` for metadata whose lifetime should follow object keys (when appropriate).
- Close DB clients, sockets, file streams and background jobs during shutdown.

## 9. Common mistakes
Believing GC fixes retained Maps; increasing `--max-old-space-size` rather than finding the leak; ignoring Buffer memory; capturing heap snapshots with sensitive data; treating natural V8 heap growth as a leak.

## Practice
Build a test route that appends objects to a global array, hit it repeatedly, graph `heapUsed`, then replace it with a bounded queue and repeat.

**Interview answer:** Node.js memory leaks commonly come from objects reachable through long-lived Maps, timers, listeners or closures. Diagnose using repeated load, memory trends and heap snapshots, then remove/bound references and verify that post-GC memory stabilizes.
