# Lesson 6 — Single-Threaded Nature of Node.js

## The Famous Statement

You often hear:

> Node.js is single-threaded.

This is only partially correct.

A better statement is:

> JavaScript execution in a standard Node.js process primarily runs on one main thread, while Node.js can use other threads and operating-system mechanisms for background work.

## Main Thread

Your JavaScript normally executes on one main thread.

```js
console.log("A");
console.log("B");
console.log("C");
```

These statements execute sequentially.

## But Node.js Itself Is Not Just One Thread

Node.js may involve:
- libuv thread pool
- OS networking threads/mechanisms
- Worker Threads
- child processes
- native libraries

## Mental Model

```text
                 Node.js Process

            Main JavaScript Thread
                     |
                 Event Loop
                     |
          +----------+----------+
          |                     |
          v                     v
   Operating System      libuv Thread Pool
   networking, etc.      selected operations
```

## Why One Main JavaScript Thread?

A single execution thread avoids many shared-memory synchronization problems.

In multi-threaded application code, developers often deal with:
- locks
- mutexes
- race conditions
- deadlocks

Node's event-driven model simplifies many I/O-heavy use cases.

## The Danger

CPU-heavy JavaScript blocks everything running on the main thread.

Example:

```js
import http from "node:http";

http.createServer((req, res) => {
  const start = Date.now();

  while (Date.now() - start < 5000) {}

  res.end("Done");
}).listen(3000);
```

During that loop, other requests cannot be processed normally by the same event loop.

## I/O Heavy vs CPU Heavy

Node.js excels at:

```text
API call -> wait
DB query -> wait
Redis -> wait
Socket -> wait
File -> wait
```

Node.js struggles when the main thread continuously computes:

```text
CPU CPU CPU CPU CPU CPU CPU
```

## Solutions for CPU Work

Use:
- Worker Threads
- child processes
- job queues
- external computation services
- horizontally scaled workers

## Interview Question

### If Node.js is single-threaded, how can it handle thousands of connections?

Because connections spend much of their lifetime waiting on I/O. Node.js delegates asynchronous work and uses the event loop to process callbacks when work completes instead of blocking one application thread per connection.

## Interview Trap

Question:

"Does Node.js use only one thread?"

Best answer:

No. JavaScript execution typically runs on one main thread, but Node.js internally uses additional threads and operating-system mechanisms. libuv maintains a thread pool for certain operations, and developers can explicitly create Worker Threads.

## Summary

```text
Single JavaScript execution thread
            ≠
Node.js uses only one thread
```

This distinction is one of the most important foundations for understanding Node.js correctly.
