# Lesson 5 — Node.js Runtime Architecture

## Architecture Overview

Node.js is made of multiple layers working together.

```text
+----------------------------------+
|       Your JavaScript Code       |
+----------------------------------+
                |
                v
+----------------------------------+
|      Node.js Core JavaScript     |
| fs | http | stream | crypto ...  |
+----------------------------------+
                |
                v
+----------------------------------+
|          C/C++ Bindings          |
+----------------------------------+
          /             \
         v               v
+-------------+      +-------------+
|     V8      |      |    libuv    |
+-------------+      +-------------+
                          |
                          v
                   Operating System
```

## Layer 1 — Application Code

This is what you write:

```js
import http from "node:http";
```

## Layer 2 — Node Core APIs

Node provides built-in modules including:
- fs
- http
- path
- crypto
- stream
- events
- os

These APIs form the developer-facing runtime.

## Layer 3 — Native Bindings

Some Node.js capabilities require communication between JavaScript and lower-level C/C++ libraries.

Bindings connect these layers.

## Layer 4 — V8

V8 executes JavaScript.

It handles:
- parsing
- compilation
- execution
- garbage collection
- heap management

## Layer 5 — libuv

libuv provides major asynchronous runtime capabilities.

It handles or coordinates:
- event loop
- asynchronous I/O
- filesystem operations
- networking abstractions
- timers
- thread pool

## Layer 6 — Operating System

Ultimately, real work is performed using OS facilities:

Linux examples:
- epoll
- file descriptors
- sockets
- processes
- filesystem syscalls

Other operating systems use different mechanisms.

## Why This Architecture Is Important

When debugging performance, you need to know which layer is responsible.

Example:

```text
Slow endpoint
   |
   +--> JavaScript CPU work?
   +--> Database?
   +--> Network?
   +--> Thread pool saturation?
   +--> File I/O?
   +--> Memory pressure?
```

A professional Node.js developer does not treat all performance problems as "Node is slow."

## Industry Example

Suppose password hashing becomes slow under heavy traffic.

bcrypt may use thread-pool-backed work depending on the implementation.

If many expensive operations compete for limited worker threads, unrelated operations may also slow down.

Understanding runtime architecture helps you diagnose this.

## Interview Question

### What are the major internal parts of Node.js?

A strong answer should mention:
- V8
- Node core APIs
- native bindings
- libuv
- operating-system interfaces
- event loop
- thread pool

## Summary

Node.js is a runtime stack, not just V8.

```text
JavaScript
   ↓
Node Core
   ↓
Bindings
   ↓
V8 + libuv
   ↓
Operating System
```

This architecture is the foundation for understanding event loop behavior, asynchronous I/O, performance, worker threads, and scalability.
