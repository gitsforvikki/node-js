# Lesson 178 — Event Loop Blocking

## 1. What is blocking?
The **event loop** runs JavaScript callbacks on the main Node.js thread. **Blocking** means one task occupies that thread so long that other callbacks cannot run promptly.

Imagine one cashier serving customers: if a customer asks for a complicated calculation at the counter, everyone else waits—even those who need a tiny purchase.

## 2. Reproduce the problem
Save as `blocking.js`:
```js
import http from "node:http";

const server = http.createServer((req, res) => {
  if (req.url === "/slow") {
    const start = Date.now();
    while (Date.now() - start < 5000) {} // intentionally BAD
    return res.end("Slow task completed");
  }
  res.end("Fast response");
});
server.listen(3000, () => console.log("Listening on 3000"));
```
Call `curl http://localhost:3000/slow`, then in another terminal call `curl http://localhost:3000/`. The second request can wait until the 5-second loop finishes.

## 3. A timer does NOT block the thread
```js
import { setTimeout as delay } from "node:timers/promises";

app.get("/slow-io-simulation", async (req, res) => {
  await delay(5000);
  res.send("Done");
});
```
This route waits 5 seconds, but **does not consume the JS thread during the timer wait**. Other requests can proceed.

## 4. Other blocking operations
- `fs.readFileSync()` on a large file.
- `crypto.pbkdf2Sync()`, `zlib.gzipSync()` for heavy input.
- Expensive regex processing (ReDoS).
- Very large `JSON.parse()` or `JSON.stringify()`.
- Huge loops and recursive calculations.

Synchronous filesystem methods are especially risky inside HTTP request handlers.

## 5. Async does not equal nonblocking CPU work
```js
app.get("/still-blocks", async (req, res) => {
  let x = 0;
  for (let i = 0; i < 200_000_000; i++) x += i;
  res.json({ x });
});
```
The `async` keyword only changes how the function returns and handles Promises. The loop still blocks.

## 6. How to detect it
Use `monitorEventLoopDelay()` (Lesson 176), CPU profiles (Lesson 182), latency p95/p99, and event-loop utilization.

```js
import { performance } from "node:perf_hooks";
let previous = performance.eventLoopUtilization();
setInterval(() => {
  const delta = performance.eventLoopUtilization(previous);
  previous = performance.eventLoopUtilization();
  console.log("Event loop utilization:", delta.utilization.toFixed(2));
}, 5000).unref();
```
High utilization together with increasing latency suggests a busy main thread; it isn't definitive proof of which function is responsible.

## 7. Solutions
1. Replace sync I/O with async APIs where appropriate.
2. Optimize slow algorithms.
3. Move heavy CPU JS to worker threads or dedicated job workers.
4. Partition expensive tasks and yield to later turns using `setImmediate` if useful.
5. Bound incoming payload sizes and expensive user-driven searches.
6. Measure improvement under concurrent requests.

## 8. Chunked work example
```js
import { setImmediate as yieldTurn } from "node:timers/promises";
async function sumLargeRange(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) {
    sum += i;
    if (i % 50_000 === 0) await yieldTurn();
  }
  return sum;
}
```
Chunks allow the event loop to process other work, but still consume CPU overall and have scheduling overhead. For genuinely heavy repeated work, worker threads are usually a better fit.

## 9. Mistakes
Assuming Node is always nonblocking; assuming Promises mean another thread; measuring only CPU percentage; blindly increasing thread-pool size; running sync I/O in busy routes.

## Practice
Compare response time of `/slow` and `/` in the same process. Replace the busy loop with `await delay(5000)`; explain why the second request changes.

**Interview answer:** Event-loop blocking happens when long-running synchronous JavaScript prevents the main loop from servicing other callbacks. Diagnose with latency/event-loop metrics and CPU profiles; optimize, yield in chunks, or offload CPU work.
