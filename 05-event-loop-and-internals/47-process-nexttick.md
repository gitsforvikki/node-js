# Lesson 47 — process.nextTick()

## Why process.nextTick matters

`process.nextTick()` is a Node.js-specific scheduling API.

It is commonly tested because it behaves differently from:

- Promise microtasks
- setTimeout
- setImmediate

Understanding it helps explain Node.js execution priority.

---

## 1. Basic example

```js
console.log("A");

process.nextTick(() => {
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

The callback runs after the current synchronous code finishes.

---

## 2. nextTick vs Promise

```js
console.log("start");

Promise.resolve().then(() => {
  console.log("promise");
});

process.nextTick(() => {
  console.log("nextTick");
});

console.log("end");
```

Typical output:

```text
start
end
nextTick
promise
```

In Node.js, the nextTick queue has higher priority than regular Promise microtasks.

---

## 3. Important terminology

The name `nextTick` is misleading if you imagine:

> next event-loop iteration

That is not the best mental model.

A better interpretation:

> run this callback immediately after the current JavaScript operation completes, before the event loop continues normally.

---

## 4. Why it exists

Historically, `process.nextTick` is useful when an API wants to defer a callback until after the current call stack but still run it before I/O/timers.

Example:

```js
function connect(callback) {
  process.nextTick(() => {
    callback(null, "connected");
  });
}
```

This ensures asynchronous callback behavior even if the result is already available.

---

## 5. Consistent async API behavior

Imagine:

```js
function getUser(id, callback) {
  if (cache.has(id)) {
    callback(null, cache.get(id));
    return;
  }

  db.get(id, callback);
}
```

Problem:

- cached result calls callback synchronously
- DB result calls callback asynchronously

This creates inconsistent behavior.

Better conceptually:

```js
if (cache.has(id)) {
  process.nextTick(() => {
    callback(null, cache.get(id));
  });

  return;
}
```

Now both paths are asynchronous.

---

## 6. nextTick starvation

This is the main danger.

```js
function repeat() {
  process.nextTick(repeat);
}

repeat();
```

What happens?

```text
nextTick callback
    |
    v
schedule nextTick
    |
    v
nextTick callback
    |
    v
schedule nextTick
    |
    v
event loop cannot progress
```

Timers and I/O may starve.

---

## 7. Why nextTick has such high priority

Node.js processes the nextTick queue before continuing through normal event-loop phases.

This makes it useful for:
- deferred API callbacks
- cleanup immediately after current stack
- compatibility patterns

But dangerous for:
- recursive scheduling
- large workloads

---

## 8. nextTick vs setImmediate

```text
process.nextTick
   -> runs before event loop continues

setImmediate
   -> runs in check phase
```

Example:

```js
process.nextTick(() => {
  console.log("nextTick");
});

setImmediate(() => {
  console.log("immediate");
});
```

Expected:

```text
nextTick
immediate
```

---

## 9. nextTick vs setTimeout

```js
process.nextTick(() => {
  console.log("nextTick");
});

setTimeout(() => {
  console.log("timeout");
}, 0);
```

Expected:

```text
nextTick
timeout
```

---

## 10. Modern usage guidance

Do not use `process.nextTick` simply because you want something "fast."

Prefer:
- Promise microtasks for Promise-based flows
- setImmediate when you want to yield to the event loop
- normal async APIs for I/O

Use nextTick when its specific semantics are actually needed.

---

## 11. Common mistakes

### Mistake 1

Thinking nextTick means next event-loop cycle.

Wrong.

### Mistake 2

Using recursive nextTick loops.

Can starve I/O.

### Mistake 3

Replacing all setImmediate or Promise scheduling with nextTick.

This can create unfair scheduling.

---

## 12. Interview questions

### What is process.nextTick?

A Node.js API that schedules a callback to run after the current operation completes but before the event loop continues to normal phases.

### Which runs first: nextTick or Promise.then?

Typically `process.nextTick`.

### Why can nextTick be dangerous?

Recursive or excessive use can starve the event loop and delay I/O/timers.

---

## 13. Strong interview answer

> process.nextTick is a Node.js-specific scheduling API. Its callbacks run after the current synchronous stack but before the event loop continues to normal phases, and before standard Promise microtasks. It is useful for deferring work while preserving very high priority, but recursive nextTick usage can starve I/O and timers.

---

## Interview-Ready Summary

```text
process.nextTick
   |
   +--> after current stack
   +--> before Promise microtasks
   +--> before normal event-loop phases
   +--> Node.js specific

Danger:
starvation
```

## Practice Task

Predict:

```js
console.log("1");

process.nextTick(() => {
  console.log("2");

  process.nextTick(() => {
    console.log("3");
  });
});

Promise.resolve().then(() => {
  console.log("4");
});

console.log("5");
```
