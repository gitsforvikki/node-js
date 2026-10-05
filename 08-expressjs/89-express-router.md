# Lesson 89 — Express Router

## Why Express Router matters

`express.Router()` is essential for structuring medium and large Express applications.

A Router is like a mini Express application focused on a route group.

---

## 1. Create a router

```js
import {
  Router,
} from "express";

const router =
  Router();
```

---

## 2. Define routes

```js
router.get(
  "/",
  listUsers
);

router.post(
  "/",
  createUser
);

router.get(
  "/:id",
  getUser
);
```

---

## 3. Export router

```js
export default router;
```

---

## 4. Mount router

```js
import userRouter
  from "./routes/user.routes.js";

app.use(
  "/api/users",
  userRouter
);
```

Final routes:

```text
GET  /api/users
POST /api/users
GET  /api/users/:id
```

---

## 5. Why Router is useful

Without Router:

```text
app.js
  100+ route definitions
```

With Router:

```text
routes/
  user.routes.js
  order.routes.js
  auth.routes.js
  admin.routes.js
```

This improves modularity.

---

## 6. Router-level middleware

```js
router.use(
  authenticate
);
```

All later routes in that router pass through authentication.

---

## 7. Route-specific middleware

```js
router.post(
  "/",
  authenticate,
  validateCreateUser,
  createUser
);
```

This is useful when only one route needs a particular rule.

---

## 8. Nested routers

Example:

```text
/users/:userId/orders
```

Parent:

```js
userRouter.use(
  "/:userId/orders",
  orderRouter
);
```

Child router:

```js
const orderRouter =
  Router({
    mergeParams: true,
  });
```

Now:

```js
req.params.userId
```

can be available inside the child router.

---

## 9. Router composition

A scalable API router:

```js
const apiRouter =
  Router();

apiRouter.use(
  "/users",
  userRouter
);

apiRouter.use(
  "/orders",
  orderRouter
);

apiRouter.use(
  "/auth",
  authRouter
);

app.use(
  "/api/v1",
  apiRouter
);
```

---

## 10. Separation of responsibilities

Route file:

```text
method
path
middleware
controller
```

Controller:

```text
HTTP translation
```

Service:

```text
business logic
```

Repository:

```text
data access
```

This keeps routes easy to scan.

---

## 11. Bad router design

Bad:

```js
router.post(
  "/",
  async (req, res) => {
    // validate
    // auth
    // query DB
    // payment
    // email
    // analytics
    // response
  }
);
```

Too much logic in one place.

---

## 12. Good router design

```js
router.post(
  "/",
  authenticate,
  validateCreateOrder,
  createOrder
);
```

Simple and declarative.

---

## 13. Router factory pattern

Sometimes routers need dependencies.

Example concept:

```js
export function createUserRouter(
  userController
) {
  const router =
    Router();

  router.get(
    "/",
    userController.list
  );

  return router;
}
```

This can improve testability and dependency injection.

---

## 14. Route naming consistency

Keep predictable resource conventions.

```text
/users
/orders
/payments
/subscriptions
```

Avoid inconsistent mixtures such as:

```text
/getUser
/orders/create
/remove-payment
```

unless intentionally using RPC-style routes.

---

## 15. Router security boundary

Example:

```js
const adminRouter =
  Router();

adminRouter.use(
  authenticate
);

adminRouter.use(
  requireRole(
    "admin"
  )
);
```

Everything below becomes an explicit admin surface.

This is a strong architecture pattern.

---

## 16. Router-specific 404

You can optionally add a fallback inside a router.

Example:

```js
router.use(
  (req, res) => {
    res.status(404).json({
      message:
        "User route not found",
    });
  }
);
```

Usually a global 404 is enough unless route groups need specialized behavior.

---

## 17. Common mistakes

### Mistake 1
One giant router.

### Mistake 2
Business logic inside route files.

### Mistake 3
Forgetting `mergeParams` in nested routers when required.

### Mistake 4
Duplicating common middleware on every route.

### Mistake 5
Poor route naming.

---

## 18. Interview questions

### What is express.Router?

A modular mini-router used to group related routes and middleware.

### Why use Router?

To split routes by resource and improve modularity, security boundaries, and maintainability.

### What is mergeParams?

It allows a child router to access parent route parameters.

### Can routers have middleware?

Yes.

---

## 19. Strong interview answer

> express.Router is a modular routing abstraction that lets me group related endpoints and middleware independently from the main app. I typically create separate routers for users, orders, auth, and admin areas, keep route files declarative, and mount them under a shared prefix. Nested routers can use mergeParams when they need parent path parameters.

---

## Interview-Ready Summary

```text
express.Router()
    |
    +--> routes
    +--> middleware
    +--> nested routers
    +--> route groups
    +--> security boundaries

Good route file:
method + path + middleware + controller
```

## Practice Task

Create:

```text
/api/v1/users
/api/v1/orders
/api/v1/admin
```

using separate routers and nested middleware.
