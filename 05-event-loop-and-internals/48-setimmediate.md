# Lesson 48 — setImmediate()

## What is setImmediate?

`setImmediate()` schedules a callback to execute in the event loop's **check phase**.

Example:

```js
setImmediate(() => {
  console.log("immediate");
});
```

---

## 1. Why setImmediate exists

Sometimes you want to:

- finish the current work
- allow the event loop to progress
- run a callback soon afterward

`setImmediate` is designed for that kind of scheduling.

---

## 2. Event-loop position

Simplified:

```text
poll phase
   |
   v
check phase
   |
   v
setImmediate callbacks
```

That connection to the check phase is the most important interview fact.

---

## 3. setImmediate is asynchronous

```js
console.log("A");

setImmediate(() => {
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

## 4. setImmediate inside I/O

This is where setImmediate becomes especially predictable.

```js
import fs from "node:fs";

fs.readFile("data.txt", () => {
  console.log("I/O complete");

  setImmediate(() => {
    console.log("immediate");
  });
});
```

After the poll-phase I/O callback finishes, the loop proceeds to check, where the immediate callback runs.

---

## 5. setImmediate vs process.nextTick

```js
setImmediate(() => {
  console.log("immediate");
});

process.nextTick(() => {
  console.log("nextTick");
});
```

Typical output:

```text
nextTick
immediate
```

Because nextTick runs before normal phase progression.

---

## 6. Yielding long work

One practical use of setImmediate is breaking a large job into smaller chunks so the event loop can breathe.

Bad:

```js
for (let i = 0; i < 1e9; i++) {
  doWork(i);
}
```

This blocks the event loop.

Chunked approach:

```js
function processChunk(start) {
  const end = Math.min(start + 1000, items.length);

  for (let i = start; i < end; i++) {
    doWork(items[i]);
  }

  if (end < items.length) {
    setImmediate(() => {
      processChunk(end);
    });
  }
}
```

This yields between chunks.

---

## 7. Important limitation

Chunking CPU work with setImmediate improves responsiveness, but it does not create real CPU parallelism.

For heavy computation, better tools may be:

- Worker Threads
- child processes
- background workers

---

## 8. setImmediate returns a handle

```js
const handle = setImmediate(() => {
  console.log("run");
});
```

Cancel:

```js
clearImmediate(handle);
```

---

## 9. Common mistakes

### Mistake 1

Thinking setImmediate means "execute immediately."

Wrong.

It means schedule for the check phase.

### Mistake 2

Using setImmediate for CPU parallelism.

It does not move work to another thread.

### Mistake 3

Assuming it always runs before setTimeout(0).

Not always at top level.

Context matters.

---

## 10. Interview questions

### Where does setImmediate run?

In the check phase of the Node.js event loop.

### Why is setImmediate useful after I/O?

Because I/O callbacks commonly execute during poll, and the event loop proceeds to check afterward.

### Does setImmediate run on another thread?

No.

### Can setImmediate reduce event-loop blocking?

It can help split long work into chunks and yield between them, but it does not provide true parallel computation.

---

## 11. Strong interview answer

> setImmediate schedules a callback for the check phase of the Node.js event loop. It is especially useful after I/O callbacks because the loop typically proceeds from poll to check, making setImmediate a natural way to run work immediately after I/O. It can also be used to yield between chunks of work, although it does not create parallel CPU execution.

---

## Interview-Ready Summary

```text
setImmediate
   |
   +--> check phase
   +--> asynchronous
   +--> useful after I/O
   +--> can yield between chunks

Not:
   immediate synchronous execution
   CPU parallelism
```

## Practice Task

Create a loop that processes a large array in chunks using `setImmediate`.

Measure how the server responsiveness differs from one huge synchronous loop.
