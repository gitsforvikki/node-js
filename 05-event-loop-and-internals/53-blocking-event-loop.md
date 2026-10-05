# Lesson 53 — Blocking the Event Loop

## Why this lesson matters

Blocking the event loop is one of the most dangerous performance problems in a Node.js server.

A single slow JavaScript operation can delay:

- every incoming request
- timers
- promise continuations
- socket callbacks
- health checks
- graceful shutdown logic

---

## 1. What does "blocking the event loop" mean?

It means the main JavaScript thread is busy for too long and cannot return control to the event loop.

Example:

```js
const start = Date.now();

while (Date.now() - start < 5000) {
  // blocking
}
```

For 5 seconds, no other JavaScript callback can run.

---

## 2. Why one request can hurt everyone

Example:

```js
app.get("/heavy", (req, res) => {
  const result = calculateHugeResult();

  res.json(result);
});
```

Timeline:

```text
Request A
   |
   v
heavy calculation
   |
   | 5 seconds
   |
Request B waits
Request C waits
Request D waits
timer waits
socket callback waits
```

This creates head-of-line blocking.

---

## 3. Common causes

Typical event-loop blockers include:

- huge loops
- CPU-heavy parsing
- synchronous filesystem APIs
- synchronous crypto APIs
- large JSON serialization/parsing
- catastrophic regular expressions
- image/video processing
- compression done synchronously
- large in-memory transformations

---

## 4. Sync filesystem example

Bad:

```js
app.get("/file", (req, res) => {
  const data =
    fs.readFileSync("large.csv");

  res.send(data);
});
```

The event loop cannot process other callbacks during the read.

---

## 5. Sync crypto example

Bad:

```js
const key = crypto.pbkdf2Sync(...);
```

In a request handler, this can block the main thread.

Prefer async equivalents when appropriate.

---

## 6. JSON can block too

Developers often forget that JSON operations are synchronous.

```js
const data = JSON.parse(hugePayload);
```

or:

```js
JSON.stringify(hugeObject);
```

For very large payloads, this can create noticeable latency.

---

## 7. Regex danger

A poorly designed regular expression can cause catastrophic backtracking.

Example concept:

```text
small malicious input
   |
   v
regex burns CPU for seconds
   |
   v
event loop blocked
```

This can become a denial-of-service risk.

This category is often called ReDoS.

---

## 8. Detecting event-loop blocking

Useful signals include:

- high request latency
- high event-loop delay
- CPU spikes
- timers firing late
- health checks timing out
- low throughput under CPU load

Monitoring event-loop delay is valuable in production.

---

## 9. Event-loop lag

Conceptually:

```js
const expected = Date.now() + 100;

setTimeout(() => {
  const delay = Date.now() - expected;

  console.log(delay);
}, 100);
```

If delay is large, the event loop may have been busy.

In production, use proper metrics rather than homemade timing alone.

---

## 10. How to fix blocking work

Possible solutions:

### Use async APIs

Replace synchronous filesystem/crypto APIs with async equivalents.

### Use streams

For large files or payloads, process incrementally.

### Use Worker Threads

For CPU-heavy JavaScript.

### Use child processes

For isolated workloads or external commands.

### Use background jobs

Move slow work out of the request lifecycle.

### Chunk work

Use `setImmediate` or similar yielding patterns for moderate workloads.

---

## 11. Chunking example

Bad:

```js
for (const item of hugeArray) {
  expensive(item);
}
```

Chunked:

```js
function processChunk(index = 0) {
  const end = Math.min(
    index + 1000,
    hugeArray.length
  );

  for (let i = index; i < end; i++) {
    expensive(hugeArray[i]);
  }

  if (end < hugeArray.length) {
    setImmediate(() => {
      processChunk(end);
    });
  }
}
```

This yields between chunks.

---

## 12. But chunking is not always enough

For truly CPU-intensive work, chunking still uses the main thread.

Better:

```text
Main Thread
   |
   +--> request handling

Worker Thread
   |
   +--> CPU-heavy calculation
```

---

## 13. Security perspective

Event-loop blocking can become a denial-of-service attack.

Examples:

- maliciously huge JSON
- expensive regex input
- expensive decompression
- oversized request bodies

Mitigations:

- request-size limits
- validation
- timeouts
- rate limits
- safe regex patterns
- resource caps

---

## 14. Production example

Imagine:

```text
POST /generate-report
```

Bad design:

- fetch 1M rows
- JSON stringify huge result
- create PDF synchronously
- respond

Better architecture:

```text
API request
   |
   v
enqueue report job
   |
   v
worker processes report
   |
   v
store result
   |
   v
notify user
```

This protects request latency.

---

## 15. Interview questions

### What blocks the Node.js event loop?

Long-running synchronous JavaScript or synchronous native APIs executed on the main thread.

### Why is event-loop blocking dangerous?

Because one blocking operation delays all other JavaScript callbacks in that process.

### How do you fix CPU-heavy work?

Use Worker Threads, child processes, background jobs, or separate services.

### Does using async keyword prevent blocking?

No.

---

## 16. Strong interview answer

> Event-loop blocking occurs when the main JavaScript thread stays busy for too long, usually because of CPU-heavy synchronous code or synchronous APIs. Since the event loop cannot schedule other callbacks while the stack is busy, one request can increase latency for every request in the process. The usual solutions are asynchronous APIs for I/O, streams for large data, Worker Threads or separate processes for CPU-heavy work, and moving expensive work to background jobs.

---

## Interview-Ready Summary

```text
Blocking causes:
  huge loops
  sync I/O
  sync crypto
  huge JSON
  regex backtracking
  CPU-heavy transforms

Symptoms:
  latency
  timer delay
  CPU high
  health checks slow

Fix:
  async APIs
  streams
  Worker Threads
  jobs
  chunking
```

## Practice Task

Build two routes:

```text
/block
/yield
```

`/block` should run a long synchronous loop.

`/yield` should process work in chunks with `setImmediate`.

Send concurrent requests and compare behavior.
