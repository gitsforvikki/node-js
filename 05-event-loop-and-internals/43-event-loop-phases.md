# Lesson 43 — Event Loop Phases

## Why this lesson matters

Interviewers often ask:

> What are the phases of the Node.js event loop?

A weak answer only lists phase names.

A strong answer explains:
- what each phase does
- where timers run
- where setImmediate runs
- where I/O callbacks are processed
- how microtasks interact with phases
- why execution order can change depending on context

---

## 1. High-level event loop phases

A simplified Node.js event loop cycle looks like:

```text
+---------------------------+
|          timers           |
+---------------------------+
|    pending callbacks      |
+---------------------------+
|       idle / prepare      |
+---------------------------+
|           poll            |
+---------------------------+
|          check            |
+---------------------------+
|      close callbacks      |
+---------------------------+
```

Not all phases are equally relevant to application developers.

The most important are:

- timers
- poll
- check
- close callbacks

---

## 2. Timers phase

This phase executes callbacks scheduled by:

- `setTimeout()`
- `setInterval()`

Example:

```js
setTimeout(() => {
  console.log("timer");
}, 100);
```

Important:

The 100 ms means:

> do not run before approximately this threshold

It does NOT mean:

> execute exactly at 100 ms

If the event loop is busy, execution happens later.

---

## 3. Pending callbacks phase

This phase handles certain system-level callbacks deferred from previous loop iterations.

Most application developers do not interact with this phase directly.

Interview note:

You should know it exists, but do not overcomplicate its purpose.

---

## 4. Idle / prepare

These are internal libuv phases.

Application code normally does not schedule callbacks directly here.

In interviews, mention them briefly and move on.

---

## 5. Poll phase

The poll phase is extremely important.

It handles many I/O-related callbacks.

Conceptually:

```text
network I/O
filesystem completion
socket events
other I/O callbacks
        |
        v
      poll
```

The poll phase may:
- execute ready I/O callbacks
- wait for new I/O events when appropriate
- decide whether to continue to check/timers depending on pending work

---

## 6. Check phase

This is where `setImmediate()` callbacks execute.

Example:

```js
setImmediate(() => {
  console.log("immediate");
});
```

Mental model:

```text
poll
  |
  v
check
  |
  v
setImmediate callbacks
```

---

## 7. Close callbacks phase

This handles close events such as some socket/handle closures.

Example concept:

```js
socket.on("close", () => {
  console.log("socket closed");
});
```

---

## 8. Microtasks are not a normal event loop phase

This is very important.

Promise callbacks are not a separate official libuv phase.

Node.js processes microtasks at specific points around JavaScript callback execution.

Important microtask-like queues include:

- `process.nextTick()` queue
- Promise microtask queue

---

## 9. process.nextTick vs Promise microtasks

Example:

```js
console.log("A");

process.nextTick(() => {
  console.log("B");
});

Promise.resolve().then(() => {
  console.log("C");
});

console.log("D");
```

Typical output:

```text
A
D
B
C
```

In Node.js, the `process.nextTick` queue is processed before the regular Promise microtask queue.

---

## 10. Timer vs immediate

At top level:

```js
setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});
```

The order can be influenced by timing and environment details.

Do not memorize:

```text
setTimeout always first
```

That is not a reliable rule.

---

## 11. Inside I/O callback

This case is more predictable.

```js
import fs from "node:fs";

fs.readFile("data.txt", () => {
  setTimeout(() => {
    console.log("timeout");
  }, 0);

  setImmediate(() => {
    console.log("immediate");
  });
});
```

Inside an I/O callback, `setImmediate` commonly executes before `setTimeout(..., 0)`.

Why?

Because after poll, the loop moves to check before returning to a future timers phase.

Mental model:

```text
I/O callback runs in poll
        |
        v
schedule timeout
schedule immediate
        |
        v
check phase
        |
        v
setImmediate
        |
        v
next loop timers
        |
        v
setTimeout
```

---

## 12. Promise inside timer callback

```js
setTimeout(() => {
  console.log("timer");

  Promise.resolve().then(() => {
    console.log("promise");
  });
}, 0);

setTimeout(() => {
  console.log("timer2");
}, 0);
```

The microtask created inside the first timer callback is typically processed before moving on to the next callback.

Conceptually:

```text
timer callback
   |
   v
promise microtask
   |
   v
next callback
```

This is important for execution-order questions.

---

## 13. Event loop phase mental model

```text
Timers
   |
Pending
   |
Poll
   |
Check
   |
Close
   |
repeat

Between callback executions:
   |
   +--> nextTick queue
   +--> Promise microtasks
```

This is a better interview model than imagining one single global callback queue.

---

## 14. Why phase knowledge matters in production

You normally do not manually optimize around phases.

But the knowledge helps with:

- debugging execution order
- understanding timers
- understanding setImmediate
- diagnosing starvation
- writing low-level infrastructure code
- answering interview puzzles correctly

---

## 15. Avoid overfitting to timing puzzles

A professional answer should not be:

> Node.js is just timers first, then promises, then I/O.

That is too simplistic and often wrong.

Better:

> Node.js processes work through multiple event loop phases, while nextTick and Promise microtasks are drained at specific boundaries around callback execution.

---

## 16. Version-awareness

Node.js event loop timing behavior can evolve because Node.js and libuv change over time.

For interviews, focus on stable conceptual rules:

- timers have threshold-based scheduling
- poll handles I/O readiness
- check runs setImmediate
- microtasks have higher priority than proceeding to later phase callbacks
- process.nextTick is especially high priority in Node.js

---

## 17. Interview questions

### What are the main event loop phases?

Timers, pending callbacks, idle/prepare, poll, check, and close callbacks.

### Where does setImmediate execute?

The check phase.

### Where do setTimeout callbacks execute?

The timers phase.

### Is the Promise microtask queue an event loop phase?

No. Microtasks are processed around callback execution and phase progression.

### Why can setImmediate beat setTimeout(0) inside I/O?

Because an I/O callback commonly runs in the poll phase, after which the loop reaches check before a future timers phase.

---

## 18. Strong interview answer

> The Node.js event loop is divided into phases including timers, pending callbacks, poll, check, and close callbacks. Timers execute setTimeout and setInterval callbacks, poll handles many I/O callbacks, and check executes setImmediate callbacks. In addition to these phases, Node.js drains the process.nextTick queue and Promise microtask queue at specific boundaries. This is why execution order cannot be explained using a single callback queue.

---

## Interview-Ready Summary

```text
Timers
  -> setTimeout
  -> setInterval

Poll
  -> I/O callbacks

Check
  -> setImmediate

Close
  -> close events

Microtasks
  -> not a phase
  -> Promise callbacks

process.nextTick
  -> higher priority than Promise microtasks
```

## Practice Task

Predict the output:

```js
import fs from "node:fs";

fs.readFile("data.txt", () => {
  console.log("I/O");

  setTimeout(() => {
    console.log("timeout");
  }, 0);

  setImmediate(() => {
    console.log("immediate");
  });

  Promise.resolve().then(() => {
    console.log("promise");
  });
});
```

Explain the order using poll, check, timers and microtasks.
