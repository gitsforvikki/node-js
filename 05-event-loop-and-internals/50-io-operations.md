# Lesson 50 — How I/O Operations Work

## Why this lesson matters

Node.js is often described as:

> non-blocking I/O

But what does that actually mean?

To answer well in interviews, you should understand how Node.js handles:

- network I/O
- filesystem I/O
- DNS
- thread-pool-backed work
- OS-level async operations

---

## 1. What is I/O?

I/O means input/output.

Examples:

- reading a file
- writing a file
- making an HTTP request
- receiving socket data
- database communication
- DNS lookup

I/O usually involves waiting for something external.

---

## 2. Why waiting matters

Suppose a database query takes 200 ms.

The CPU does not need to actively calculate for all 200 ms.

Most of the time is waiting.

Blocking approach:

```text
JavaScript
   |
   v
send DB query
   |
   v
wait 200 ms
   |
   v
continue
```

Non-blocking approach:

```text
send DB query
   |
   +----------------------+
   |                      |
   v                      |
handle other work         |
                          |
                DB result arrives
                          |
                          v
                 callback becomes ready
```

---

## 3. Node.js does not use one mechanism for all I/O

This is a key interview point.

Different operations may use different underlying mechanisms.

Examples:

```text
Network sockets
  -> OS async event mechanisms

Filesystem
  -> often libuv thread pool

Some DNS
  -> thread-pool-backed or OS facilities

Crypto
  -> selected operations use thread pool
```

---

## 4. Network I/O

Network sockets are usually handled efficiently by the operating system using event notification mechanisms.

Examples:
- epoll on Linux
- kqueue on BSD/macOS-like systems
- IOCP on Windows

Conceptually:

```text
Node.js
   |
   v
register socket interest
   |
   v
Operating System
   |
   | waits efficiently
   |
   v
socket becomes ready
   |
   v
libuv notified
   |
   v
event loop callback
```

Node.js does not need one JavaScript thread per socket.

---

## 5. Filesystem I/O

Filesystem async APIs are commonly implemented using libuv's worker thread pool because portable async filesystem interfaces are not uniform across operating systems.

Example:

```js
fs.readFile("data.txt", callback);
```

Conceptually:

```text
JavaScript
   |
   v
fs.readFile
   |
   v
libuv
   |
   v
worker thread
   |
   v
filesystem
   |
   v
completion
   |
   v
event loop callback
```

---

## 6. Thread pool is limited

The libuv thread pool has a finite size.

That means heavy thread-pool-backed operations can compete with each other.

Examples:
- filesystem work
- some DNS operations
- some crypto operations
- compression

If the pool is saturated:

```text
Task 1 -> worker
Task 2 -> worker
Task 3 -> worker
Task 4 -> worker
Task 5 -> waits
Task 6 -> waits
```

This can create latency even though your JavaScript looks asynchronous.

---

## 7. Async does not mean infinite capacity

This is very important.

```js
await Promise.all(
  hugeArray.map(readFile)
);
```

Even though everything is async, the system still has limits:

- thread pool
- file descriptors
- memory
- sockets
- database connections

Async improves utilization, not physical capacity.

---

## 8. Database I/O

Database clients usually communicate through network sockets.

Conceptually:

```text
Node API
   |
   v
database driver
   |
   v
TCP socket
   |
   v
DB server
   |
   v
response
   |
   v
socket readable
   |
   v
event loop
```

The database itself executes the query elsewhere.

---

## 9. DNS nuance

DNS behavior varies by API.

Some DNS APIs may use OS facilities directly, while others may involve the libuv thread pool.

This is one reason advanced Node.js developers avoid oversimplified statements like:

> all async I/O uses the thread pool

That is false.

---

## 10. I/O completion flow

A good mental model:

```text
JavaScript starts operation
        |
        v
Node/native layer
        |
        v
OS or libuv worker
        |
        v
operation completes
        |
        v
completion reported to libuv
        |
        v
callback becomes eligible
        |
        v
event loop schedules callback
        |
        v
JavaScript handles result
```

---

## 11. Blocking I/O vs non-blocking I/O

Blocking:

```js
const data = fs.readFileSync("data.txt");
```

Non-blocking:

```js
fs.readFile("data.txt", callback);
```

The key difference is whether the main JavaScript thread has to wait.

---

## 12. Why Node.js scales well for I/O-heavy systems

A typical API request may look like:

```text
10 ms JavaScript
150 ms database wait
5 ms JavaScript
80 ms API wait
5 ms JavaScript
```

Most time is waiting.

Node.js can use that waiting time to process other requests.

---

## 13. Production example

Suppose 1,000 users call an API.

Each request waits on the database.

Node.js does not create 1,000 JavaScript threads.

Instead:

```text
many sockets
   |
   v
OS handles readiness
   |
   v
event loop handles callbacks
```

This is one of Node.js's strongest architectural advantages.

---

## 14. I/O bottlenecks still exist

Potential bottlenecks:

- DB connection pool
- slow remote service
- thread-pool saturation
- network bandwidth
- file descriptor limits
- memory
- rate limits

The event loop cannot magically remove external bottlenecks.

---

## 15. Common mistakes

### Mistake 1

Saying all async operations use the thread pool.

Wrong.

### Mistake 2

Thinking async I/O has unlimited capacity.

Wrong.

### Mistake 3

Using synchronous I/O in hot request paths.

Blocks event loop.

### Mistake 4

Blaming Node.js when the real bottleneck is DB or network.

Always profile the full system.

---

## 16. Interview questions

### How does Node.js perform non-blocking I/O?

It delegates waiting work to the operating system, libuv, or libuv's thread pool, then processes completion callbacks through the event loop.

### Does all async I/O use the thread pool?

No. Network I/O often relies on OS event mechanisms, while filesystem and some crypto/DNS operations may use the libuv thread pool.

### Why is Node.js good for network servers?

Because network operations spend significant time waiting, and Node.js can coordinate many sockets without blocking one JavaScript thread per connection.

---

## 17. Strong interview answer

> Node.js handles I/O using different underlying mechanisms. Network sockets are typically managed through operating-system event notification facilities, while filesystem and some crypto or DNS operations may use libuv's thread pool. JavaScript initiates the operation and continues. When the operation completes, libuv reports readiness and the event loop schedules the corresponding callback. This is what allows Node.js to handle many concurrent I/O operations efficiently without blocking the main JavaScript thread.

---

## Interview-Ready Summary

```text
I/O
  -> mostly waiting

Node.js delegates to:
  -> OS async mechanisms
  -> libuv
  -> thread pool

Network
  -> usually OS event readiness

Filesystem
  -> commonly thread pool

Important:
async != unlimited resources
all async != thread pool
```

## Section 5 Progress Map

```text
Event Loop
   |
   +--> phases
   +--> call stack
   +--> task queues
   +--> microtasks
   +--> process.nextTick
   +--> setImmediate
   +--> timers
   +--> I/O internals
```

The next lessons go deeper into:
- libuv
- thread pool
- event-loop blocking
- interview execution problems

## Practice Task

Write down which mechanism you expect for:

1. HTTP socket read
2. file read
3. database query
4. password hashing
5. DNS lookup

Then explain which ones may involve the OS event system vs the libuv thread pool.
