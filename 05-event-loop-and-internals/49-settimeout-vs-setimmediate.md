# Lesson 49 — setTimeout() vs setImmediate()

## Why this comparison is famous

This is a classic Node.js interview question.

The key is not memorizing one fixed order.

The real answer is:

> Their order depends on where and when they are scheduled.

---

## 1. setTimeout

```js
setTimeout(() => {
  console.log("timeout");
}, 0);
```

Runs through the timers mechanism after the minimum delay threshold has been reached.

---

## 2. setImmediate

```js
setImmediate(() => {
  console.log("immediate");
});
```

Runs in the check phase.

---

## 3. Top-level comparison

```js
setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});
```

Do not confidently claim one always runs first.

At top level, timing may depend on runtime state and event-loop timing.

A strong interview answer explicitly mentions this nuance.

---

## 4. Inside I/O callback

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

Typical order:

```text
immediate
timeout
```

Why?

```text
poll phase
   |
   | I/O callback
   |
   +--> schedule timer
   +--> schedule immediate
   |
   v
check phase
   |
   v
setImmediate
   |
   v
later timers phase
   |
   v
setTimeout
```

---

## 5. setTimeout(0) does not mean 0 ms exact execution

It means roughly:

> execute no earlier than the timer threshold, once the event loop can process it.

Example:

```js
setTimeout(() => {
  console.log("timer");
}, 0);

const start = Date.now();

while (Date.now() - start < 3000) {}
```

The timer runs after the blocking loop.

---

## 6. setImmediate semantic meaning

Think:

```text
setTimeout(fn, 0)
  -> schedule through timers

setImmediate(fn)
  -> schedule for check phase
```

They are different mechanisms, not two names for the same thing.

---

## 7. nextTick vs both

```js
process.nextTick(() => {
  console.log("nextTick");
});

setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});
```

nextTick runs before both.

---

## 8. Promise microtasks vs both

```js
Promise.resolve().then(() => {
  console.log("promise");
});

setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});
```

Promise microtasks are processed before the event loop proceeds to these phase callbacks.

---

## 9. Which one should you use?

Use `setTimeout` when:

- you want a delay
- scheduling after a time threshold matters
- implementing backoff/retry delays

Use `setImmediate` when:

- you want to schedule work for the check phase
- you want to run work after current I/O processing
- you want to yield and continue shortly

---

## 10. Production examples

### setTimeout

Retry delay:

```js
await new Promise((resolve) => {
  setTimeout(resolve, 1000);
});
```

### setImmediate

Chunked processing:

```js
setImmediate(processNextChunk);
```

---

## 11. Common mistakes

### Mistake 1

Assuming setTimeout(0) means immediate execution.

Wrong.

### Mistake 2

Memorizing "setTimeout always before setImmediate."

Wrong.

### Mistake 3

Using setImmediate for real delays.

Use timers.

### Mistake 4

Using either one to solve heavy CPU work permanently.

For heavy CPU tasks, use appropriate parallelism tools.

---

## 12. Interview questions

### Which runs first: setTimeout(0) or setImmediate?

At top level, order is not guaranteed purely by syntax. Inside many I/O callbacks, setImmediate commonly runs first because the loop moves from poll to check before a future timers phase.

### Where does setTimeout run?

Timers phase.

### Where does setImmediate run?

Check phase.

---

## 13. Strong interview answer

> setTimeout schedules a callback after a minimum delay threshold and is processed through the timers phase, while setImmediate schedules a callback for the check phase. Their top-level order is not something you should treat as universally fixed. However, when both are scheduled inside an I/O callback, setImmediate commonly executes first because the event loop moves from poll to check before reaching a later timers phase.

---

## Interview-Ready Summary

```text
setTimeout
   -> timers
   -> delay threshold

setImmediate
   -> check phase
   -> useful after I/O

Inside I/O:
   setImmediate commonly first

Top level:
   do not assume fixed order
```

## Practice Task

Run the comparison:
1. at top level
2. inside `fs.readFile`
3. inside another timer callback

Explain why context matters.
