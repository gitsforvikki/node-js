# Lesson 44 — Call Stack

## Why the call stack matters

The call stack is the foundation of JavaScript execution.

If you understand it, you can explain:

- synchronous execution
- nested function calls
- stack traces
- recursion
- stack overflow
- why async callbacks wait
- how the event loop decides when JavaScript can continue

---

## 1. What is the call stack?

The call stack is a LIFO data structure used by the JavaScript engine to keep track of currently executing function calls.

LIFO means:

```text
Last In
First Out
```

---

## 2. Simple example

```js
function first() {
  second();
}

function second() {
  third();
}

function third() {
  console.log("hello");
}

first();
```

Call stack:

```text
first()
   |
   v
second()
   |
   v
third()
   |
   v
console.log()
```

Internally:

```text
Top
+-------------+
| console.log |
+-------------+
| third       |
+-------------+
| second      |
+-------------+
| first       |
+-------------+
Bottom
```

As functions finish, they are popped off the stack.

---

## 3. Stack flow

```text
push first
push second
push third
push console.log

pop console.log
pop third
pop second
pop first
```

This explains synchronous execution order.

---

## 4. Why async callbacks wait

Example:

```js
setTimeout(() => {
  console.log("timer");
}, 0);

function heavy() {
  const start = Date.now();

  while (Date.now() - start < 2000) {}
}

heavy();
```

The timer callback cannot execute while `heavy()` is still on the call stack.

```text
Call Stack

heavy()
  |
  | blocking
  |
  v
stack finally empty
  |
  v
event loop can schedule timer callback
```

This is one of the most important links between call stack and event loop.

---

## 5. Global execution context

When a JavaScript file starts, top-level code runs in a global/module execution context.

Conceptually:

```text
main script
   |
   +--> function A
   |      |
   |      +--> function B
   |
   +--> function C
```

The stack tracks which execution context is active.

---

## 6. Stack traces

When an error occurs:

```js
function a() {
  b();
}

function b() {
  c();
}

function c() {
  throw new Error("Boom");
}

a();
```

The stack trace may show:

```text
Error: Boom
    at c (...)
    at b (...)
    at a (...)
```

This is a direct reflection of the call chain.

Understanding stack traces is a practical debugging skill.

---

## 7. Stack overflow

Recursive code can overflow the stack.

Example:

```js
function recurse() {
  recurse();
}

recurse();
```

Eventually:

```text
RangeError: Maximum call stack size exceeded
```

Why?

Every recursive call adds another stack frame.

```text
recurse
recurse
recurse
recurse
...
```

Memory allocated for the call stack is finite.

---

## 8. Recursion vs loop

Recursive:

```js
function countdown(n) {
  if (n === 0) return;

  countdown(n - 1);
}
```

Each nested call adds a frame.

Iterative:

```js
for (let i = n; i > 0; i--) {
  // work
}
```

A loop does not create a new function call frame per iteration.

---

## 9. Synchronous exception propagation

Example:

```js
function a() {
  b();
}

function b() {
  throw new Error("Failure");
}

try {
  a();
} catch (error) {
  console.log("caught");
}
```

The error propagates back up the current call stack.

```text
b throws
   |
   v
a
   |
   v
try/catch catches
```

---

## 10. Async stack boundary

Now:

```js
try {
  setTimeout(() => {
    throw new Error("Failure");
  }, 0);
} catch {
  console.log("not caught");
}
```

The timer callback runs later on a new call stack.

The original try/catch frame is gone.

This is why async errors require different handling.

---

## 11. Promise continuation creates later execution

```js
async function run() {
  console.log("before");

  await Promise.resolve();

  console.log("after");
}
```

Before await:
- current stack

After await:
- continuation runs later

Conceptually:

```text
run()
   |
   v
"before"
   |
   v
await
   |
   v
current stack unwinds

later:

run continuation
   |
   v
"after"
```

---

## 12. Call stack vs heap

Do not confuse them.

```text
Call Stack
  -> function calls
  -> local execution frames
  -> control flow

Heap
  -> objects
  -> arrays
  -> dynamically allocated data
```

Example:

```js
function createUser() {
  const user = {
    name: "Vikash",
  };

  return user;
}
```

The variable reference is part of the execution frame, while the object itself lives in heap-managed memory.

---

## 13. Long stack blocking

The problem is not just how many frames exist.

The bigger production issue is how long JavaScript keeps the stack busy.

Example:

```js
function expensive() {
  for (let i = 0; i < 5e9; i++) {}
}
```

One long-running frame can block the event loop.

---

## 14. Interview questions

### What is the call stack?

A LIFO structure used by the JavaScript engine to track active function calls and execution contexts.

### Why can't an async callback run while synchronous code is still executing?

Because JavaScript callback execution waits until the current call stack is free.

### What causes "Maximum call stack size exceeded"?

Usually uncontrolled or excessively deep recursion that creates too many stack frames.

### What is the difference between stack and heap?

The stack tracks function execution frames; the heap stores dynamically allocated objects and data.

---

## 15. Strong interview answer

> The call stack is a LIFO structure used by V8 to track currently executing JavaScript functions. Every function call pushes a stack frame, and returning pops it. Asynchronous callbacks cannot execute until the current stack is free, which is why CPU-heavy synchronous code blocks the event loop. Stack traces reflect this call chain, and deep recursion can cause a maximum call stack error.

---

## Interview-Ready Summary

```text
Call Stack
   |
   +--> LIFO
   +--> function execution
   +--> stack frames
   +--> synchronous control flow
   +--> stack traces
   +--> recursion

Event Loop rule:
callbacks wait until stack is free
```

## Practice Task

Draw the stack manually for:

```js
function one() {
  two();
}

function two() {
  three();
}

function three() {
  console.log("done");
}

one();
```

Then explain what changes if `two()` schedules `three()` using `setTimeout`.
