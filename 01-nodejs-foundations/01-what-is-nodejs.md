# Lesson 1 — What is Node.js?

## What you'll learn
- What Node.js actually is
- Why it became popular for backend development
- Where Node.js fits in modern application architecture
- Where Node.js is a great choice—and where it is not
- The mental model interviewers expect

## Core Concept

Node.js is a **JavaScript runtime** that allows JavaScript to run outside the browser.

JavaScript itself is only a programming language. A browser normally provides the environment that executes it. Node.js provides a different environment optimized for servers, CLI tools, scripts, networking, file systems, and backend applications.

A useful mental model is:

```text
JavaScript
   |
   v
V8 JavaScript Engine
   |
   v
Node.js Runtime
   |
   +--> File System
   +--> Networking
   +--> Processes
   +--> Streams
   +--> Timers
   +--> Operating System APIs
```

## What Node.js is NOT

Node.js is not:
- a programming language
- a framework
- a database
- a web server framework
- the same thing as Express

Express is a framework that runs **on top of Node.js**.

## Why Node.js Became Popular

Node.js became popular because backend development often involves waiting:

- waiting for database responses
- waiting for APIs
- waiting for files
- waiting for network packets
- waiting for Redis
- waiting for message queues

Instead of blocking a thread for each waiting operation, Node.js uses an event-driven, non-blocking I/O model.

That makes it especially effective for high-concurrency I/O-heavy systems.

## Example

```js
console.log("Start");

setTimeout(() => {
  console.log("Timer finished");
}, 1000);

console.log("End");
```

Output:

```text
Start
End
Timer finished
```

Node.js does not stop the entire program while waiting for the timer.

## Real-World Use Cases

Node.js is commonly used for:
- REST APIs
- GraphQL servers
- authentication services
- real-time chat
- WebSocket applications
- API gateways
- microservices
- background workers
- CLI tools
- server-side rendering
- build tooling

## When Node.js is a Great Choice

Choose Node.js when your application is mostly:
- I/O-heavy
- API-driven
- event-driven
- real-time
- highly concurrent

Examples:

```text
Chat App
    |
    +--> Messages
    +--> Presence
    +--> Notifications
    +--> WebSockets
```

## When Node.js May Not Be the Best Choice

CPU-intensive work can block the main JavaScript thread.

Examples:
- video encoding
- heavy image processing
- large scientific calculations
- CPU-heavy encryption loops
- machine learning computation

These tasks can still be handled using:
- Worker Threads
- child processes
- external services
- distributed jobs

## Industry Mental Model

Think of Node.js as:

> A JavaScript runtime designed around event-driven, non-blocking I/O that is excellent for concurrent network applications.

## Common Mistakes

### Mistake 1
"Node.js is single-threaded, therefore it can only do one thing."

Wrong.

JavaScript execution is primarily single-threaded, but Node.js can coordinate many asynchronous operations through the event loop, operating system, libuv, and thread pool.

### Mistake 2
"Node.js is a backend framework."

Wrong.

Node.js is the runtime. Express, Fastify, NestJS, and Koa are frameworks/libraries built on top of it.

## Interview Questions

### What is Node.js?
Node.js is an open-source JavaScript runtime built around the V8 engine that allows JavaScript to execute outside the browser and provides APIs for networking, file systems, processes, streams, and other server-side operations.

### Why is Node.js good for scalable APIs?
Because its non-blocking, event-driven architecture allows it to handle many concurrent I/O operations efficiently without creating a dedicated application thread for each request.

### Is Node.js single-threaded?
JavaScript execution is primarily single-threaded, but Node.js itself is not limited to one thread internally. libuv, the OS, worker threads, and thread pools may use additional threads.

## Interview-Ready Summary

```text
Node.js
  = JavaScript runtime
  + V8
  + Node APIs
  + libuv
  + event-driven architecture
  + non-blocking I/O
```

Node.js is especially strong for **I/O-heavy, concurrent, network-based applications**.

## Practice Task

Create a file named `app.js`:

```js
console.log("Node.js is running");

console.log({
  platform: process.platform,
  nodeVersion: process.version,
  pid: process.pid,
});
```

Run:

```bash
node app.js
```

Observe how Node.js gives JavaScript access to information that browser JavaScript normally does not expose.
