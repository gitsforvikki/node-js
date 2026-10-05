# Lesson 56 — Creating Custom Events

## Why custom events matter

Custom events help decouple application components.

Instead of one service directly calling every side effect, it can publish an event and allow separate listeners to react.

Example:

```text
Order Service
    |
    v
order.created
    |
    +--> Email Listener
    +--> Analytics Listener
    +--> Audit Listener
```

---

## 1. Extending EventEmitter

A clean pattern is to create your own event bus.

```js
import {
  EventEmitter,
} from "node:events";

class AppEvents extends EventEmitter {}

export const appEvents =
  new AppEvents();
```

---

## 2. Emit custom events

```js
appEvents.emit(
  "user.created",
  {
    id: 1,
    email: "vikash@example.com",
  }
);
```

---

## 3. Listen to custom events

```js
appEvents.on(
  "user.created",
  (user) => {
    console.log(
      "Welcome email:",
      user.email
    );
  }
);
```

---

## 4. Event naming conventions

Use consistent names.

Good:

```text
user.created
user.updated
order.created
payment.completed
payment.failed
```

Avoid vague names:

```text
done
changed
success
event1
```

Good event names describe something that already happened.

---

## 5. Events should usually describe facts

Better:

```text
payment.completed
```

Less ideal:

```text
completePayment
```

Why?

Commands describe intent.

Events describe facts.

```text
Command:
  createOrder

Event:
  order.created
```

This distinction is useful in event-driven architecture.

---

## 6. Keep payloads clear

Example:

```js
appEvents.emit(
  "order.created",
  {
    orderId: order.id,
    userId: order.userId,
    total: order.total,
  }
);
```

Avoid passing huge mutable objects unnecessarily.

A focused payload makes listeners more stable.

---

## 7. Avoid exposing internal mutable state

Bad:

```js
appEvents.emit(
  "order.created",
  orderDocument
);
```

If listeners mutate the same object, behavior can become difficult to reason about.

Prefer a stable event payload.

---

## 8. Register listeners during startup

Good:

```js
registerUserEventListeners();
registerOrderEventListeners();

startServer();
```

Bad:

```js
app.post("/orders", (req, res) => {
  appEvents.on(
    "order.created",
    handler
  );

  // ...
});
```

The bad version adds another listener on every request.

That can create memory leaks and duplicate side effects.

---

## 9. Custom event module structure

```text
src/
├── events/
│   ├── event-bus.js
│   ├── user.listeners.js
│   └── order.listeners.js
├── services/
└── server.js
```

---

## 10. Example architecture

```js
// event-bus.js
import {
  EventEmitter,
} from "node:events";

export const eventBus =
  new EventEmitter();
```

```js
// order.listeners.js
import {
  eventBus,
} from "./event-bus.js";

export function registerOrderListeners() {
  eventBus.on(
    "order.created",
    async (event) => {
      await sendOrderEmail(event);
    }
  );
}
```

```js
// order.service.js
eventBus.emit(
  "order.created",
  {
    orderId: order.id,
    userId: order.userId,
  }
);
```

---

## 11. Important async-listener problem

This looks safe:

```js
eventBus.on(
  "order.created",
  async () => {
    throw new Error("Email failed");
  }
);
```

But `emit()` does not automatically await returned Promises.

That means async listener rejections need deliberate handling.

Possible strategy:

```js
eventBus.on(
  "order.created",
  (event) => {
    void sendEmail(event).catch(
      (error) => {
        logger.error(error);
      }
    );
  }
);
```

---

## 12. Critical workflow warning

Suppose:

```text
Order Created
   |
   +--> send email
   +--> charge money
```

Charging money should probably not be a casual in-memory event listener.

Why?

If the process crashes after order creation but before the payment listener runs, the payment action may be lost.

Critical workflows need stronger guarantees:
- transaction
- queue
- outbox pattern
- durable messaging

---

## 13. Domain events

In larger architectures, custom events may represent domain facts:

```text
UserRegistered
OrderPlaced
PaymentSucceeded
SubscriptionExpired
```

This can improve separation between:
- core business logic
- notifications
- analytics
- auditing

---

## 14. In-process event bus trade-offs

Advantages:

- simple
- fast
- loose coupling
- easy to add listeners

Disadvantages:

- in-memory only
- no retry
- no durability
- process-local
- listener failures require careful handling

---

## 15. Common mistakes

### Mistake 1
Registering listeners repeatedly.

### Mistake 2
Using events for critical guaranteed workflows.

### Mistake 3
Ignoring async listener rejection.

### Mistake 4
Using vague event names.

### Mistake 5
Passing huge mutable objects.

---

## 16. Interview questions

### Why create custom events?

To decouple producers from side-effect consumers inside an application.

### Where should listeners be registered?

Usually during application startup, not inside request handlers.

### Are custom EventEmitter events durable?

No.

### What happens if an async listener rejects?

EventEmitter does not automatically await or centrally handle that Promise.

---

## 17. Strong interview answer

> Custom events are useful for decoupling in-process components. A service can emit a domain fact such as order.created, while separate listeners handle email, analytics, or auditing. Listeners should generally be registered once during startup. Because EventEmitter is in-memory and does not provide durability or retries, critical workflows should use a queue or durable event system instead.

---

## Interview-Ready Summary

```text
Custom Events
   |
   +--> domain fact
   +--> loose coupling
   +--> startup registration
   +--> focused payload
   +--> async errors handled explicitly

Use for:
  email
  analytics
  audit

Avoid for:
  guaranteed distributed workflows
```

## Practice Task

Create a reusable application event bus with:

- `user.registered`
- `order.created`
- `payment.failed`

Register all listeners once during startup and ensure async listener failures are logged safely.
