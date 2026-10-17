# Lesson 208 — Layered Architecture

## 1. Meaning
**Layered architecture** groups code by responsibility. A layer talks to another through a clear interface rather than knowing every implementation detail.

Restaurant analogy: customer → waiter → kitchen → inventory. A customer doesn't directly manage kitchen storage.

```text
Presentation layer (HTTP / controllers)
          |
Business layer (services / policies)
          |
Data access layer (repositories)
          |
Infrastructure (MongoDB, Redis, email providers)
```

## 2. End-to-end request example
A user cancels an order:
1. Route checks authentication and validates order ID.
2. Controller passes authenticated user ID and order ID to service.
3. Service checks whether cancellation is allowed.
4. Repository performs a **conditional update** to prevent race conditions.
5. Controller translates result to HTTP status and safe JSON.

```js
// service logic, independent of Express
async function cancelOrder(userId, orderId, orders) {
  const updated = await orders.cancelIfPending({ userId, orderId });
  if (!updated) throw new Error("Order not found or cannot be cancelled");
  return updated;
}
```

```js
// MongoDB repository boundary
const orders = {
  cancelIfPending: ({ userId, orderId }) =>
    Order.findOneAndUpdate(
      { _id: orderId, userId, status: "PENDING" },
      { $set: { status: "CANCELLED" } },
      { new: true }
    ),
};
```
A conditional write avoids a check-then-update race. A real API maps expected errors to stable status codes and validates ObjectIds.

## 3. Layering vs feature folders
Layering describes **responsibilities**. Folder structure describes **where files live**. You can organize by feature while keeping these layers inside each feature:
```text
modules/orders/
  order.routes.js
  order.controller.js
  order.service.js
  order.repository.js
```

## 4. Dependency direction
If your business service imports `req` or `res`, it becomes coupled to Express. Prefer plain values like `userId` and `orderId` so the service can run from an HTTP route, queue worker, or scheduled job.

## 5. Cross-cutting concerns
Logging, authentication, validation, tracing and transactions can span multiple layers. Give them clear ownership; avoid repeating authorization checks inconsistently. Authentication is usually middleware; **resource authorization** often belongs in service/query policy.

## 6. When layering helps / hurts
Helps when rules are complicated, teams collaborate, many routes share logic and unit tests need isolation. Hurts when every one-line method is forwarded through five files without value.

## 7. Common mistakes
Controllers issuing unrelated DB queries; circular dependencies; business services returning HTTP responses; forgetting database atomicity; believing folder separation automatically creates a good architecture.

## Practice
Draw layers for `POST /checkout`. Place input validation, tax calculation, stock reservation, payment provider calls, DB queries and HTTP response in the appropriate layer.

**Interview answer:** Layering separates presentation, business logic and persistence, reducing coupling and making systems more testable. Keep dependencies clear and enforce concurrency rules at the data boundary.
