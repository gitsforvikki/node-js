# Lesson 55 — EventEmitter

## Why this lesson matters

Node.js is heavily event-driven.

Many important Node.js APIs are built around events, including:

- streams
- HTTP servers
- sockets
- process events
- file watchers
- custom application events

Understanding `EventEmitter` makes many other Node.js concepts easier.

---

## 1. What is EventEmitter?

`EventEmitter` is a class from the `node:events` module that lets objects:

- emit named events
- register listeners
- react when those events happen

Import:

```js
import { EventEmitter } from "node:events";
```

Basic example:

```js
const emitter = new EventEmitter();

emitter.on("greet", (name) => {
  console.log(`Hello ${name}`);
});

emitter.emit("greet", "Vikash");
```

Output:

```text
Hello Vikash
```

---

## 2. Mental model

```text
Emitter
   |
   | emit("orderCreated")
   v
Event
   |
   +--> Listener A
   +--> Listener B
   +--> Listener C
```

The emitter does not need to know the internal logic of each listener.

This helps decouple components.

---

## 3. on()

Registers a listener.

```js
emitter.on("message", (message) => {
  console.log(message);
});
```

Every time the event is emitted, the listener runs.

---

## 4. once()

Registers a listener that runs only once.

```js
emitter.once("connected", () => {
  console.log("Connected first time");
});
```

After the first emission, the listener is removed automatically.

---

## 5. emit()

Triggers an event.

```js
emitter.emit("connected");
```

You can pass arguments:

```js
emitter.emit(
  "userCreated",
  {
    id: 1,
    name: "Vikash",
  }
);
```

Listener:

```js
emitter.on("userCreated", (user) => {
  console.log(user.id);
});
```

---

## 6. EventEmitter listeners are synchronous

This is a key interview point.

```js
emitter.on("event", () => {
  console.log("A");
});

emitter.on("event", () => {
  console.log("B");
});

console.log("start");

emitter.emit("event");

console.log("end");
```

Output:

```text
start
A
B
end
```

Listeners run synchronously in registration order by default.

---

## 7. Async work inside listeners

A listener can schedule async work:

```js
emitter.on("event", async () => {
  await doSomething();

  console.log("done");
});
```

But the EventEmitter itself does not automatically await that Promise.

That means:

```js
emitter.emit("event");

console.log("after emit");
```

`after emit` may run before the async listener finishes.

This is extremely important.

---

## 8. Removing listeners

Store the function reference:

```js
function handler(data) {
  console.log(data);
}

emitter.on("data", handler);
```

Remove it:

```js
emitter.off("data", handler);
```

or:

```js
emitter.removeListener(
  "data",
  handler
);
```

---

## 9. removeAllListeners()

```js
emitter.removeAllListeners("data");
```

Use carefully.

You may unintentionally remove listeners registered by other parts of the application.

---

## 10. The special "error" event

This is one of the most important EventEmitter rules.

```js
emitter.emit(
  "error",
  new Error("Something failed")
);
```

If an EventEmitter emits `error` and no error listener exists, Node.js may treat it as an uncaught error and terminate the process.

Handle it when appropriate:

```js
emitter.on("error", (error) => {
  console.error(error);
});
```

---

## 11. Listener count

```js
console.log(
  emitter.listenerCount("message")
);
```

Useful for debugging and detecting unexpected listener growth.

---

## 12. MaxListenersExceededWarning

Node.js may warn if too many listeners are attached to the same event.

Example:

```text
MaxListenersExceededWarning
```

This can indicate:

- listeners added repeatedly
- listeners never removed
- lifecycle bug
- potential memory leak

Do not immediately solve it with:

```js
emitter.setMaxListeners(1000);
```

First investigate why listeners keep growing.

---

## 13. EventEmitter and streams

Streams are EventEmitters.

Example readable stream events:

```js
stream.on("data", (chunk) => {});
stream.on("end", () => {});
stream.on("error", (err) => {});
```

This is why learning EventEmitter before streams is important.

---

## 14. EventEmitter and HTTP

HTTP server objects also emit events.

Conceptually:

```text
HTTP Server
   |
   +--> request
   +--> connection
   +--> close
   +--> error
```

---

## 15. Real backend example

```js
const domainEvents =
  new EventEmitter();

domainEvents.on(
  "user.created",
  (user) => {
    console.log(
      "Send welcome email:",
      user.email
    );
  }
);

domainEvents.emit(
  "user.created",
  {
    id: 10,
    email: "vikash@example.com",
  }
);
```

This decouples user creation from email logic.

---

## 16. EventEmitter vs durable messaging

Important distinction:

```text
EventEmitter
  -> same Node.js process
  -> in-memory
  -> fast
  -> no durability
  -> no retry guarantee

Message Queue
  -> cross-process
  -> can persist messages
  -> retries
  -> distributed systems
```

Do not use EventEmitter as a replacement for BullMQ, RabbitMQ, Kafka, etc.

---

## 17. Common mistakes

### Mistake 1
Thinking listeners run asynchronously automatically.

Wrong.

### Mistake 2
Emitting `error` without a listener.

Dangerous.

### Mistake 3
Registering listeners repeatedly inside request handlers.

Can cause leaks.

### Mistake 4
Using EventEmitter for critical cross-service workflows.

Use durable messaging when delivery guarantees matter.

---

## 18. Interview questions

### What is EventEmitter?

A Node.js class that provides publish/subscribe style event handling inside a process.

### Are EventEmitter listeners synchronous?

Yes, by default they are invoked synchronously in registration order.

### What is special about the error event?

If emitted without an error listener, it can cause an uncaught exception and terminate the process.

### Is EventEmitter suitable for distributed messaging?

No.

---

## 19. Strong interview answer

> EventEmitter is Node.js's core in-process event abstraction. An emitter can publish named events using emit(), and listeners registered with on() or once() react to those events. Listeners execute synchronously by default, even though they may themselves start asynchronous work. EventEmitter is useful for decoupling components inside one process, but it is not durable messaging and should not replace a queue when retries or cross-service delivery are required.

---

## Interview-Ready Summary

```text
EventEmitter
   |
   +--> on()
   +--> once()
   +--> emit()
   +--> off()
   +--> error event
   +--> listener lifecycle

Important:
listeners are synchronous
async listeners are not awaited
EventEmitter is in-process only
```

## Practice Task

Create an EventEmitter-based order workflow with:

- `order.created`
- `order.cancelled`
- `order.paid`

Add listeners for:
- email
- audit logging
- analytics

Then test listener cleanup.
