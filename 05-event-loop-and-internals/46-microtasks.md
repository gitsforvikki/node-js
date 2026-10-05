# Lesson 46 — Microtasks

## Why microtasks matter

Microtasks explain why Promise callbacks often run before timers and other event-loop callbacks.

They are essential for understanding:

- Promise execution order
- async/await continuation
- process.nextTick interactions
- starvation
- event-loop interview puzzles

---

## 1. What is a microtask?

A microtask is a high-priority callback scheduled to run after the current synchronous JavaScript finishes, before the runtime proceeds to many later event-loop callbacks.

Common sources include:

- `Promise.then()`
- `Promise.catch()`
- `Promise.finally()`
- continuation after `await`

Example:

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("B");
});

console.log("C");
```

Output:

```text
A
C
B
```

---

## 2. Why does Promise.then run later?

Because the callback passed to `.then()` is not executed synchronously.

```js
const promise = Promise.resolve();

promise.then(() => {
  console.log("microtask");
});
```

The handler is queued as a microtask.

---

## 3. Promise executor vs Promise handler

This is a classic interview trap.

```js
console.log("1");

new Promise((resolve) => {
  console.log("2");
  resolve();
}).then(() => {
  console.log("3");
});

console.log("4");
```

Output:

```text
1
2
4
3
```

Why?

- Promise executor runs synchronously
- `.then()` runs later as a microtask

---

## 4. Microtasks and timers

```js
console.log("start");

setTimeout(() => {
  console.log("timer");
}, 0);

Promise.resolve().then(() => {
  console.log("promise");
});

console.log("end");
```

Typical output:

```text
start
end
promise
timer
```

Mental model:

```text
sync code
   |
   v
microtasks
   |
   v
timer callback
```

---

## 5. async/await uses Promise machinery

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

Output:

```text
1
A
2
B
```

The code after `await` resumes later through Promise-based microtask scheduling.

---

## 6. Microtask draining

After synchronous work finishes, the runtime processes queued microtasks.

If those microtasks schedule more microtasks, the queue may continue draining before normal event-loop callbacks proceed.

Example:

```js
Promise.resolve().then(() => {
  console.log("A");

  Promise.resolve().then(() => {
    console.log("B");
  });
});
```

Both are handled before the runtime moves on to later phase work.

---

## 7. Microtask starvation

This is important.

```js
function loop() {
  Promise.resolve().then(loop);
}

loop();
```

This continually adds new microtasks.

Possible effect:

```text
microtask
   |
   v
new microtask
   |
   v
new microtask
   |
   v
timers/I/O delayed
```

This is called starvation.

---

## 8. Microtasks inside callbacks

Example:

```js
setTimeout(() => {
  console.log("timer 1");

  Promise.resolve().then(() => {
    console.log("microtask");
  });
}, 0);

setTimeout(() => {
  console.log("timer 2");
}, 0);
```

The microtask from the first timer callback is typically processed before the next timer callback.

Conceptually:

```text
timer 1 callback
   |
   v
microtask queue
   |
   v
microtask
   |
   v
timer 2 callback
```

---

## 9. Microtask queue vs task queues

Simplified:

```text
Current JavaScript
      |
      v
Microtasks
      |
      v
Event-loop callbacks
```

But Node.js also has a special higher-priority `process.nextTick` queue, covered in the next lesson.

---

## 10. Common misconceptions

### Misconception 1

Promises are synchronous because a resolved Promise already has a value.

Wrong.

The Promise may already be fulfilled, but `.then()` still runs asynchronously.

### Misconception 2

Microtasks are an event-loop phase.

Wrong.

They are processed at specific execution boundaries, not as a normal libuv phase.

### Misconception 3

Microtasks can never hurt performance.

Wrong.

Huge or recursive microtask chains can starve other work.

---

## 11. Interview questions

### What is a microtask?

A high-priority scheduled callback that runs after the current synchronous stack completes and before many later event-loop callbacks.

### Which operations commonly create microtasks?

Promise handlers and async/await continuations.

### Why does Promise.then usually run before setTimeout(0)?

Because Promise handlers are processed as microtasks before the runtime proceeds to timer callbacks.

### Can microtasks starve the event loop?

Yes, if new microtasks are continuously scheduled.

---

## 12. Strong interview answer

> Microtasks are high-priority asynchronous callbacks used by Promises and async/await continuations. After the current synchronous stack finishes, Node.js processes microtasks before moving on to many event-loop phase callbacks such as timers. This is why Promise.then usually executes before setTimeout with zero delay. Excessive recursive microtasks can starve the event loop.

---

## Interview-Ready Summary

```text
Microtasks
   |
   +--> Promise.then
   +--> Promise.catch
   +--> Promise.finally
   +--> async/await continuation

Order:
sync code
  -> microtasks
  -> event-loop callbacks

Risk:
microtask starvation
```

## Practice Task

Predict the output:

```js
console.log("1");

Promise.resolve()
  .then(() => {
    console.log("2");
  })
  .then(() => {
    console.log("3");
  });

setTimeout(() => {
  console.log("4");
}, 0);

console.log("5");
```

Explain each step.
