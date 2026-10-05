# Lesson 54 — Event Loop Interview Problems

## Why this lesson matters

Event-loop interviews often include code-output questions.

The goal is not memorization.

You should reason using:

1. synchronous call stack
2. process.nextTick queue
3. Promise microtasks
4. timers
5. I/O callbacks
6. check phase / setImmediate

---

# Problem 1 — Promise vs Timer

```js
console.log("A");

setTimeout(() => {
  console.log("B");
}, 0);

Promise.resolve().then(() => {
  console.log("C");
});

console.log("D");
```

## Answer

```text
A
D
C
B
```

## Reason

```text
sync:
A
D

microtask:
C

timer:
B
```

---

# Problem 2 — nextTick vs Promise

```js
console.log("1");

Promise.resolve().then(() => {
  console.log("2");
});

process.nextTick(() => {
  console.log("3");
});

console.log("4");
```

## Answer

```text
1
4
3
2
```

## Reason

```text
sync
  -> 1
  -> 4

nextTick
  -> 3

Promise microtask
  -> 2
```

---

# Problem 3 — Promise executor

```js
console.log("A");

new Promise((resolve) => {
  console.log("B");

  resolve();
}).then(() => {
  console.log("C");
});

console.log("D");
```

## Answer

```text
A
B
D
C
```

## Why?

The Promise executor runs synchronously.

Only `.then()` is scheduled as a microtask.

---

# Problem 4 — async/await

```js
async function run() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("1");

run();

console.log("2");
```

## Answer

```text
1
A
2
B
```

## Reason

Everything before `await` runs synchronously.

Continuation after `await` runs as a Promise-based microtask.

---

# Problem 5 — nested microtask

```js
Promise.resolve().then(() => {
  console.log("A");

  Promise.resolve().then(() => {
    console.log("B");
  });
});

setTimeout(() => {
  console.log("C");
}, 0);
```

## Answer

```text
A
B
C
```

The newly queued microtask is drained before timers proceed.

---

# Problem 6 — nextTick inside Promise

```js
Promise.resolve().then(() => {
  console.log("promise");

  process.nextTick(() => {
    console.log("nextTick");
  });
});
```

## Reasoning

The Promise callback runs as a microtask.

Inside it, a nextTick callback is queued.

Node processes nextTick callbacks at its next applicable checkpoint before continuing normal phase work.

For interview purposes, the key is understanding:

```text
nextTick has higher scheduling priority than normal event-loop callbacks
```

Do not overgeneralize every nested ordering rule without considering runtime boundaries.

---

# Problem 7 — Timer with Promise

```js
setTimeout(() => {
  console.log("timer1");

  Promise.resolve().then(() => {
    console.log("promise");
  });
}, 0);

setTimeout(() => {
  console.log("timer2");
}, 0);
```

## Typical answer

```text
timer1
promise
timer2
```

Why?

Microtasks are drained after the first callback before the next timer callback executes.

---

# Problem 8 — setImmediate vs timeout inside I/O

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

## Typical answer

```text
immediate
timeout
```

Why?

```text
I/O callback
   |
   v
poll phase
   |
   v
check phase
   |
   v
setImmediate
   |
   v
future timers phase
   |
   v
setTimeout
```

---

# Problem 9 — CPU blocking

```js
setTimeout(() => {
  console.log("timer");
}, 0);

const start = Date.now();

while (Date.now() - start < 2000) {}

console.log("done");
```

## Answer

```text
done
timer
```

The timer becomes eligible but cannot execute until the call stack is free.

---

# Problem 10 — async function return value

```js
async function getValue() {
  return 10;
}

console.log(getValue());
```

## Answer

The function returns a Promise fulfilled with `10`.

Interview answer:

```text
async function
  -> always returns Promise
```

---

# Problem 11 — async map

```js
const values = [1, 2, 3];

const result = values.map(async (value) => {
  return value * 2;
});

console.log(result);
```

## Answer

```text
Promise[]
```

To resolve:

```js
const resolved =
  await Promise.all(result);
```

---

# Problem 12 — forEach trap

```js
const users = [1, 2, 3];

users.forEach(async (user) => {
  await saveUser(user);
});

console.log("done");
```

## Problem

`forEach` does not await the callback Promises.

"done" may print before saves finish.

---

# Problem 13 — Promise.all rejection

```js
await Promise.all([
  taskA(),
  taskB(),
  taskC(),
]);
```

Suppose `taskB` rejects.

What happens?

Answer:

- combined Promise rejects
- task A and task C are not automatically cancelled
- they may continue running

---

# Problem 14 — setTimeout vs setImmediate top-level

```js
setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});
```

## Interview-safe answer

Do not claim a universal fixed order at top level.

Timing can depend on runtime/event-loop state.

A better answer is:

> setTimeout uses the timers mechanism while setImmediate runs in the check phase. Inside I/O callbacks, setImmediate commonly executes first.

---

# Problem 15 — Complete execution puzzle

```js
console.log("1");

setTimeout(() => {
  console.log("2");
}, 0);

setImmediate(() => {
  console.log("3");
});

Promise.resolve().then(() => {
  console.log("4");
});

process.nextTick(() => {
  console.log("5");
});

console.log("6");
```

## Guaranteed beginning

```text
1
6
5
4
```

Why?

```text
sync
  -> 1
  -> 6

nextTick
  -> 5

Promise microtask
  -> 4
```

The relative order of top-level timeout and immediate should not be treated as universally fixed.

---

# How to solve event-loop questions

Use this checklist.

## Step 1 — Run synchronous code

Anything directly on the call stack runs first.

## Step 2 — Note nextTick callbacks

These have very high priority in Node.js.

## Step 3 — Note Promise microtasks

Includes:
- `.then`
- `.catch`
- `.finally`
- post-`await` continuation

## Step 4 — Identify event-loop phase callbacks

Examples:
- timers
- I/O
- setImmediate
- close

## Step 5 — After every callback, check newly queued microtasks

This is where many candidates make mistakes.

---

# Interview mental model

```text
1. synchronous stack
       |
       v
2. process.nextTick
       |
       v
3. Promise microtasks
       |
       v
4. event-loop phase callback
       |
       v
5. nextTick + microtasks again
       |
       v
6. next phase/callback
```

This is simplified but highly useful.

---

# Common interview mistakes

### Mistake 1

Memorizing outputs without understanding queues.

### Mistake 2

Treating all async callbacks equally.

### Mistake 3

Thinking `setTimeout(..., 0)` runs immediately.

### Mistake 4

Thinking Promise executor is asynchronous.

### Mistake 5

Thinking async/await has a separate runtime mechanism.

It is built on Promises.

### Mistake 6

Assuming top-level setImmediate vs setTimeout order is universally fixed.

---

# Strong interview explanation template

When asked to predict output, explain in this order:

> First, all synchronous statements execute on the call stack. Then Node.js drains the process.nextTick queue, followed by Promise microtasks. After that, the event loop continues through its phases such as timers, poll and check. If a callback schedules new microtasks, those are processed before continuing to later callbacks.

That answer shows understanding instead of memorization.

---

## Section 5 Final Mental Model

```text
JavaScript Call Stack
        |
        v
Node APIs
        |
        v
V8 + libuv
        |
        +--> event loop
        +--> OS async I/O
        +--> thread pool
        |
        v
Ready callbacks
        |
        +--> nextTick
        +--> Promise microtasks
        +--> timers
        +--> I/O
        +--> setImmediate
        |
        v
JavaScript executes callback
```

If you can explain this section clearly, you understand one of the most important foundations of Node.js.

## Practice Task

Create 10 small execution-order questions yourself using combinations of:

- synchronous logs
- Promise.resolve
- async/await
- process.nextTick
- setTimeout
- setImmediate
- fs.readFile

Predict every output before running the code and explain why.
