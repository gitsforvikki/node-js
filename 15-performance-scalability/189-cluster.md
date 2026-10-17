# Lesson 189 — Node.js Cluster Concepts

## 1. Why use clustering?
A single Node.js process normally runs application JavaScript on one main thread. A machine may have multiple CPU cores. The built-in `cluster` module starts **multiple Node.js worker processes** able to share a server port.

```text
Clients -> shared port 3000
            |
       Primary process
          /   |   \
     Worker1 Worker2 Worker3
       CPU1    CPU2    CPU3
```
Exact request-distribution strategy varies by platform/configuration. Workers are **processes**, not Worker Threads.

## 2. Runnable cluster example
```js
import cluster from "node:cluster";
import os from "node:os";
import http from "node:http";

if (cluster.isPrimary) {
  const count = Math.min(os.availableParallelism(), 4);
  console.log(`Primary PID ${process.pid}; starting ${count} workers`);

  for (let i = 0; i < count; i++) cluster.fork();

  cluster.on("exit", (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} exited`, code, signal);
    // A production supervisor should use restart backoff / crash-loop controls.
  });
} else {
  http.createServer((req, res) => {
    res.end(`Served by worker PID ${process.pid}\n`);
  }).listen(3000);
}
```
Run `node cluster.js`, then repeatedly call `curl http://localhost:3000`. Different worker PIDs may appear.

## 3. What's shared and not shared?
Workers can share the **listening port**, but they do **not** automatically share JavaScript variables, Maps, or in-memory sessions.

```js
let onlineUsers = 0; // independent value in EACH worker
```
For a shared session, rate limiter or cache, use Redis or another external store. Shared databases also need capacity planning across all workers.

## 4. Clustering vs Worker Threads
- **Cluster:** multiple Node processes, each with its own main JS event loop; helps serve multiple requests concurrently across CPU cores.
- **Worker Thread:** parallel CPU calculations in threads inside one process.
- **Horizontal scaling:** multiple app instances across hosts/containers (Lesson 190).

Clustering does not make a single CPU-heavy request automatically run across multiple cores. Each request still runs in one worker unless work is explicitly split.

## 5. Why throughput may improve
Suppose one worker uses a full CPU core. Adding workers may spread independent HTTP requests across other cores. But the benefit depends on CPU, memory, database limits and contention. Clustering is not a fix for inefficient queries or memory leaks.

## 6. Worker crashes and graceful shutdown
A robust primary process monitors exits and can replace workers with backoff. During deployment, stop routing new requests, allow in-flight work to finish, and stop workers without duplicate processing.

In production, orchestrators/PM2 may handle multi-process management, making direct `cluster` unnecessary for some deployments.

## 7. WebSockets and sticky sessions
Long-lived connections (WebSockets/Socket.IO) and in-memory rooms require special planning. Depending on transport and setup, sticky routing may be required; multi-instance Socket.IO typically needs an appropriate shared adapter such as Redis. One worker's in-memory room is not visible to another.

## 8. Common mistakes
Believing workers are threads; using in-memory session state across workers; spawning many more workers than useful cores; restarting crash loops without backoff; forgetting DB pool multiplication.

## Practice
Run the cluster demo and observe PID distribution. Add a counter in each worker and explain why it does not show the global count.

**Interview answer:** Node cluster uses multiple OS processes to share incoming connections on one port and use more CPU cores. Workers have separate memory; use shared stores and operational tooling for sessions, caching and coordination.
