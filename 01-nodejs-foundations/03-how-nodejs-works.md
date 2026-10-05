# Lesson 3 — How Node.js Works

## Big Picture

A Node.js application involves several important layers:

```text
Your JavaScript
      |
      v
     V8
      |
      v
Node.js Core APIs
      |
      v
    libuv
   /     \
 OS     Thread Pool
```

## Step 1 — JavaScript Execution

V8 parses and executes your JavaScript.

Example:

```js
const total = 10 + 20;
console.log(total);
```

This work is executed by the JavaScript engine.

## Step 2 — Node APIs

When you call:

```js
fs.readFile(...)
```

or:

```js
setTimeout(...)
```

you are using APIs supplied by Node.js.

## Step 3 — Asynchronous Work

For many asynchronous tasks, Node.js delegates work to:
- the operating system
- libuv
- libuv's thread pool

Meanwhile, JavaScript execution can continue.

## Example

```js
import fs from "node:fs";

console.log("1");

fs.readFile("data.txt", "utf8", () => {
  console.log("3");
});

console.log("2");
```

Typical output:

```text
1
2
3
```

## What Actually Happened?

```text
JS starts
   |
   v
console.log("1")
   |
   v
fs.readFile requested
   |
   +----> asynchronous system/libuv work
   |
   v
console.log("2")
   |
   v
main stack becomes free
   |
   v
file finishes
   |
   v
callback becomes eligible
   |
   v
event loop schedules callback
   |
   v
console.log("3")
```

## Why This Matters

This architecture allows one Node.js process to manage many requests that spend most of their time waiting.

Imagine 10,000 connections:

```text
Request 1 ---- waiting DB
Request 2 ---------- waiting API
Request 3 -- Redis
Request 4 ---------------- file I/O
...
```

Node.js does not need to block JavaScript execution for each waiting operation.

## Important Limitation

If JavaScript itself performs heavy CPU work:

```js
while (true) {}
```

the main JavaScript thread is blocked.

That means:
- timers stop progressing
- request callbacks cannot execute
- APIs feel frozen
- latency increases

## Industry Use

This is why production Node.js services avoid long CPU-bound work on the main thread.

Heavy work should often move to:
- Worker Threads
- queues
- dedicated workers
- separate microservices

## Interview Question

### How does Node.js handle multiple concurrent requests with a single JavaScript thread?

Node.js uses the event loop and delegates asynchronous I/O to the operating system or libuv. When operations complete, their callbacks are queued and executed when the JavaScript call stack is free.

## Summary

```text
Node.js scalability is not magic.

It comes from:
non-blocking I/O
+ event loop
+ OS async capabilities
+ libuv
+ efficient callback scheduling
```
