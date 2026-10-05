# Lesson 45 — Callback / Task Queues

## Why this lesson matters

Many developers learn an oversimplified model:

```text
Call Stack
Callback Queue
Event Loop
```

That is useful at first, but Node.js actually has multiple queues and phase-specific callback collections.

This lesson gives you an interview-ready mental model without making it unnecessarily confusing.

---

## 1. What is a task queue?

When asynchronous work completes, its callback usually does not execute immediately.

It becomes eligible for execution and waits in an appropriate queue or phase structure until JavaScript can run it.

Simplified:

```text
Async operation finishes
        |
        v
callback becomes ready
        |
        v
queue / phase
        |
        v
event loop
        |
        v
call stack
```

---

## 2. One global callback queue is too simple

In Node.js there is not just one universal callback queue.

Different kinds of callbacks are associated with different mechanisms.

Important categories include:

- timer callbacks
- I/O callbacks
- setImmediate callbacks
- close callbacks
- process.nextTick queue
- Promise microtask queue

---

## 3. Timer callbacks

Example:

```js
setTimeout(() => {
  console.log("timer");
}, 0);
```

The callback becomes eligible after the timer threshold.

It is later handled in the timers-related part of the event loop.

---

## 4. I/O callbacks

Example:

```js
fs.readFile("data.txt", () => {
  console.log("file ready");
});
```

When the I/O operation finishes, the callback becomes ready for event-loop processing.

It does not jump directly into the call stack.

---

## 5. setImmediate callbacks

```js
setImmediate(() => {
  console.log("immediate");
});
```

These are processed in the check phase.

---

## 6. Promise microtask queue

```js
Promise.resolve().then(() => {
  console.log("promise");
});
```

Promise handlers are queued as microtasks.

They have higher priority than moving on to many later event-loop callbacks.

---

## 7. process.nextTick queue

Node.js has a special `process.nextTick()` queue.

```js
process.nextTick(() => {
  console.log("nextTick");
});
```

This queue is processed before the regular Promise microtask queue.

Example:

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("promise");
});

process.nextTick(() => {
  console.log("nextTick");
});

console.log("B");
```

Typical output:

```text
A
B
nextTick
promise
```

---

## 8. Priority mental model

A useful simplified priority model is:

```text
Current synchronous code
        |
        v
process.nextTick queue
        |
        v
Promise microtasks
        |
        v
event loop phase callbacks
```

This is not the entire runtime specification, but it is an excellent interview model.

---

## 9. Example execution order

```js
console.log("start");

setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});

Promise.resolve().then(() => {
  console.log("promise");
});

process.nextTick(() => {
  console.log("nextTick");
});

console.log("end");
```

You can confidently say:

```text
start
end
nextTick
promise
...
```

The relative order of top-level zero-delay timeout and immediate can depend on runtime timing/context.

That nuance is a better answer than overconfident memorization.

---

## 10. Queue starvation

Because `process.nextTick` has very high priority, recursive usage can starve the event loop.

Example:

```js
function loop() {
  process.nextTick(loop);
}

loop();
```

This can prevent the event loop from progressing to normal I/O/timer phases.

That is dangerous.

---

## 11. Microtask starvation

Large recursive Promise chains can also delay normal event loop work.

Example concept:

```js
function loop() {
  Promise.resolve().then(loop);
}

loop();
```

Continuous microtasks can delay timers and I/O.

---

## 12. Queue does not mean separate thread

Important:

A queue stores ready work.

It does not imply a new thread.

```text
Queue
  -> waiting callbacks

Main JS Thread
  -> executes one callback at a time
```

---

## 13. Browser terminology vs Node.js

Browser discussions often use terms such as:
- task queue
- macrotask queue
- microtask queue

Node.js adds its own phase structure and `process.nextTick` semantics.

Do not blindly copy browser event loop explanations into Node.js interviews.

---

## 14. Why queue knowledge matters

It helps explain:

- promise vs timer order
- nextTick behavior
- setImmediate behavior
- starvation
- delayed callbacks
- async execution puzzles

---

## 15. Common misconceptions

### Misconception 1

> All callbacks go into one callback queue.

Wrong.

### Misconception 2

> A callback runs as soon as its async operation finishes.

Wrong.

It must wait until JavaScript can execute it.

### Misconception 3

> Promise callbacks and timer callbacks have equal priority.

Wrong.

Promise handlers are microtasks and are processed earlier at relevant boundaries.

### Misconception 4

> process.nextTick means next event-loop iteration.

Misleading.

It is processed before the event loop proceeds to the next normal phase.

---

## 16. Interview questions

### What is a callback queue?

A structure holding callbacks that are ready to execute but are waiting for the JavaScript thread/event loop to schedule them.

### Does Node.js have only one callback queue?

No. Node.js has phase-specific callback handling, plus separate process.nextTick and Promise microtask queues.

### Which runs first: nextTick or Promise.then?

In Node.js, `process.nextTick` callbacks are processed before normal Promise microtasks.

### Can queues starve the event loop?

Yes. Recursive nextTick or excessive microtask scheduling can delay timers and I/O.

---

## 17. Strong interview answer

> In Node.js, ready asynchronous callbacks do not all enter one global queue. Timers, I/O callbacks, setImmediate, and close callbacks are associated with different event-loop phases. In addition, Node.js maintains a process.nextTick queue and Promise microtask queue. After synchronous JavaScript finishes, nextTick callbacks are processed before Promise microtasks, and then the runtime continues through event-loop phases. Understanding these queues explains execution order and starvation issues.

---

## Interview-Ready Summary

```text
Ready async work
   |
   +--> timer callbacks
   +--> I/O callbacks
   +--> setImmediate callbacks
   +--> close callbacks
   +--> nextTick queue
   +--> Promise microtasks

Priority model:
sync code
  -> nextTick
  -> Promise microtasks
  -> phase callbacks
```

## Section 5 Progress Map

```text
Event Loop
   |
   +--> phases
   |
   +--> call stack
   |
   +--> callback/task queues
   |
   +--> next lessons:
          microtasks
          process.nextTick
          setImmediate
          timers
          libuv
          thread pool
          blocking
```

## Practice Task

Predict and explain:

```js
console.log("1");

process.nextTick(() => {
  console.log("2");
});

Promise.resolve().then(() => {
  console.log("3");
});

setTimeout(() => {
  console.log("4");
}, 0);

console.log("5");
```

Explain the result using:
- call stack
- nextTick queue
- Promise microtask queue
- timers phase
