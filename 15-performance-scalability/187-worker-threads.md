# Lesson 187 — Worker Threads

## 1. What is a Worker Thread?
A **Worker Thread** runs JavaScript in a separate thread within the same Node.js process. The main thread can keep accepting requests while a worker performs a CPU-heavy calculation.

```text
Main thread (HTTP/event loop)
     | postMessage(data)
     v
Worker thread (own JS event loop + V8 isolate)
     | postMessage(result)
     v
Main thread sends response
```

**Each worker has its own JavaScript heap/isolate**. Ordinary JavaScript objects aren't shared automatically; data is copied using structured cloning, transferred, or explicitly shared using `SharedArrayBuffer`.

## 2. When should we use workers?
Good: expensive data analysis, CPU-heavy JS transformations, large calculations, image-processing *JavaScript* algorithms.
Usually unnecessary: normal async DB queries, HTTP fetches, reading files via asynchronous APIs. These already have nonblocking I/O mechanisms.

## 3. Runnable example: compute on another thread
**worker.js**
```js
import { parentPort, workerData } from "node:worker_threads";

function sumTo(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) sum += i;
  return sum;
}

const result = sumTo(workerData.n);
parentPort.postMessage(result);
```

**server.js**
```js
import express from "express";
import { Worker } from "node:worker_threads";

const app = express();
function calculateInWorker(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL("./worker.js", import.meta.url), {
      workerData: { n },
    });
    worker.once("message", resolve);
    worker.once("error", reject);
    worker.once("exit", (code) => {
      if (code !== 0) reject(new Error(`Worker exited: ${code}`));
    });
  });
}

app.get("/calculate", async (req, res) => {
  const n = Number(req.query.n);
  if (!Number.isSafeInteger(n) || n < 0 || n > 50_000_000) {
    return res.status(400).json({ message: "Invalid n" });
  }
  res.json({ result: await calculateInWorker(n) });
});

app.get("/health", (req, res) => res.json({ ok: true }));
app.listen(3000);
```

Try requesting `/calculate?n=50000000` and `/health` simultaneously. The main server can handle the health request while calculation runs elsewhere. Exact timing varies with CPU contention.

## 4. Flow explained
1. Express receives the calculation request.
2. `new Worker()` creates another JavaScript execution thread.
3. `workerData` sends bounded input.
4. Worker calculates and sends a result using `postMessage()`.
5. Promise resolves; Express returns JSON.
6. The worker terminates after finishing.

## 5. Worker pools
**Creating a worker for every request is expensive** and can overwhelm memory/CPU during spikes. Production APIs use a **bounded worker pool**, a job queue, cancellation/timeouts, and backpressure. Libraries like Piscina can manage worker pools; review current compatibility.

## 6. Worker Threads vs libuv pool
Worker Threads execute **your JavaScript** on parallel threads. The libuv worker pool processes certain native async tasks (some filesystem, DNS, crypto, zlib operations); `async/await` doesn't send your JS loop there.

## 7. Error handling and cancellation
Handle worker `message`, `error`, and `exit`. Validate inputs, restrict worker concurrency, and consider `worker.terminate()` on deadlines. A terminated worker may lose partially completed work, so design side effects carefully.

## 8. Common mistakes
Using one new worker per unbounded HTTP request, assuming objects share references, using workers for ordinary DB I/O, ignoring the cost of serialization, forgetting worker failures.

## Practice
Replace a synchronous heavy Express route with a worker. Load-test latency and compare responsiveness. Then explain why a pool is better than launching 1,000 workers.

**Interview answer:** Worker Threads are for CPU-intensive JavaScript executed in parallel while the main event loop stays responsive. Use bounded pools and message passing; I/O-bound tasks usually need async I/O, not worker threads.
