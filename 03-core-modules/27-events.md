# Lesson 27 — events

## Core Concept

Node.js is heavily event-driven.

The `events` module provides the `EventEmitter` class.

Import:

```js
import { EventEmitter } from "node:events";
```

## Basic Example

```js
const emitter = new EventEmitter();

emitter.on("userCreated", (user) => {
  console.log("New user:", user);
});

emitter.emit("userCreated", {
  id: 1,
  name: "Vikash",
});
```

## Mental Model

```text
Emitter
   |
   | emit("orderCreated")
   v
Event
   |
   +--> send email
   +--> update analytics
   +--> create audit log
```

This reduces direct coupling.

## on()

Registers a listener.

```js
emitter.on("paymentSuccess", handler);
```

## once()

Runs only once.

```js
emitter.once("connected", () => {
  console.log("Connected first time");
});
```

## Removing Listeners

```js
emitter.off("event", handler);
```

or:

```js
emitter.removeListener("event", handler);
```

## Listener Arguments

```js
emitter.on("message", (roomId, message) => {
  console.log(roomId, message);
});

emitter.emit(
  "message",
  "room-1",
  "Hello"
);
```

## EventEmitter Is Synchronous

This is important.

When you call:

```js
emitter.emit("task");
```

registered listeners are invoked synchronously in registration order unless the listeners themselves schedule async work.

Example:

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

## The error Event

An `error` event deserves special attention.

```js
emitter.emit("error", new Error("Failure"));
```

If there is no error listener, EventEmitter behavior can terminate the process by throwing the error.

Handle it when appropriate:

```js
emitter.on("error", (error) => {
  console.error(error);
});
```

## Listener Leaks

If you keep adding listeners without removing them, memory can grow.

Node may warn:

```text
MaxListenersExceededWarning
```

This often indicates:
- repeated listener registration
- lifecycle bugs
- memory leak risk

Do not simply increase the listener limit without investigating.

## Industry Example

Bad coupling:

```text
createOrder()
  -> save order
  -> send email
  -> send notification
  -> analytics
  -> audit log
```

Event-driven:

```text
createOrder()
  -> save order
  -> emit orderCreated

orderCreated
  +-> email listener
  +-> analytics listener
  +-> audit listener
```

This can improve separation, but in-process EventEmitter is not a durable message queue.

If the process crashes, the event is gone.

## EventEmitter vs Message Queue

```text
EventEmitter
  same process
  in-memory
  fast
  not durable

Message Queue
  cross-process/service
  can be durable
  retries possible
  distributed architecture
```

## Interview Questions

### Are EventEmitter listeners asynchronous?

No. EventEmitter invokes listeners synchronously by default.

### Can EventEmitter replace Kafka/RabbitMQ/BullMQ?

No. It is an in-process event mechanism, not a durable distributed messaging system.

## Summary

```text
EventEmitter
  on()
  once()
  emit()
  off()
  error events
  synchronous listener execution
```

## Practice Task

Create an order service that emits:
- `orderCreated`
- `orderCancelled`

Add separate listeners for:
- email
- audit logs
- analytics
