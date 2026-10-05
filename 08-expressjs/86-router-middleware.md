# Lesson 86 — Router-level Middleware

## Why router-level middleware matters

Large Express applications should not put every middleware globally.

Router-level middleware allows you to apply behavior only to a group of related routes.

This improves:

- security boundaries
- organization
- performance
- readability

---

## 1. Basic router middleware

```js
import {
  Router,
} from "express";

const router =
  Router();

router.use(
  authenticate
);

router.get(
  "/",
  listUsers
);

router.get(
  "/:id",
  getUser
);
```

Both routes now require authentication.

---

## 2. Mounted router

```js
app.use(
  "/api/users",
  router
);
```

Flow:

```text
/api/users/*
     |
     v
router middleware
     |
     v
matching route
```

---

## 3. Why not make everything global?

Suppose:

```text
/public/*
/auth/*
/admin/*
```

If admin authorization middleware is global, it may block public endpoints.

Router-level boundaries are cleaner.

---

## 4. Protected router example

```js
const accountRouter =
  Router();

accountRouter.use(
  authenticate
);

accountRouter.get(
  "/profile",
  getProfile
);

accountRouter.get(
  "/orders",
  getOrders
);
```

Every account route is protected once.

---

## 5. Admin router

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

adminRouter.get(
  "/users",
  listUsers
);

adminRouter.delete(
  "/users/:id",
  deleteUser
);
```

This creates an explicit security boundary.

---

## 6. Router middleware order

```js
router.use(authenticate);

router.get(
  "/public-info",
  publicInfo
);
```

That route is no longer public.

If you want public routes:

```js
router.get(
  "/public-info",
  publicInfo
);

router.use(authenticate);

router.get(
  "/profile",
  getProfile
);
```

Again, order matters.

---

## 7. Path-specific router middleware

```js
router.use(
  "/admin",
  requireAdmin
);
```

Only matching router subpaths are affected.

---

## 8. Subrouters

Architecture:

```text
/api
  |
  +--> /users
  +--> /orders
  +--> /admin
```

You can compose routers:

```js
apiRouter.use(
  "/users",
  userRouter
);

apiRouter.use(
  "/orders",
  orderRouter
);

apiRouter.use(
  "/admin",
  adminRouter
);
```

This scales well.

---

## 9. Parent route parameters

When mounting nested routers with route params, child routers may need access to parent params.

Conceptually:

```text
/users/:userId/orders
```

A child order router may need `userId`.

Express Router supports configuration for merged parameters in such scenarios.

Example concept:

```js
const router =
  Router({
    mergeParams: true,
  });
```

Use when necessary and understand the resulting parameter scope.

---

## 10. Router-specific logging

Example:

```js
adminRouter.use(
  adminAuditLogger
);
```

This avoids noisy global audit behavior.

---

## 11. Router-specific rate limits

Different areas may need different rate limits.

Example:

```text
/auth/login
  -> strict rate limit

/products
  -> normal rate limit
```

Router-level middleware is a good fit.

---

## 12. Security architecture

A useful design:

```text
Public Router
   |
   +--> public endpoints

Authenticated Router
   |
   +--> authenticate
   +--> user endpoints

Admin Router
   |
   +--> authenticate
   +--> admin authorization
   +--> admin endpoints
```

---

## 13. Common mistakes

### Mistake 1
Making all middleware global.

### Mistake 2
Placing public routes after router auth middleware.

### Mistake 3
Duplicating authentication on every route instead of router-level use.

### Mistake 4
Forgetting merged params when child routers need parent params.

### Mistake 5
Creating one giant router for unrelated resources.

---

## 14. Interview questions

### What is router-level middleware?

Middleware registered on an `express.Router()` instance instead of the whole application.

### Why use it?

To apply shared behavior to a specific group of routes.

### Why does order matter?

Only routes/middleware registered after a given middleware are affected in that router flow.

### What is mergeParams for?

It allows a nested router to access parameters from its parent route.

---

## 15. Strong interview answer

> Router-level middleware applies shared behavior to a specific route group rather than the entire application. I use it to create boundaries such as authenticated user routes, admin routes, or rate-limited auth routes. This avoids duplicated middleware on individual routes and keeps security concerns close to the router they protect. Middleware registration order still determines which routes are affected.

---

## Interview-Ready Summary

```text
Router Middleware
   |
   +--> auth boundary
   +--> admin boundary
   +--> rate limit
   +--> audit logging
   +--> validation groups

Benefits:
less duplication
clearer security
better modularity
```

## Practice Task

Create:

```text
/public
/account
/admin
```

Routers where:
- public needs no auth
- account requires auth
- admin requires auth + admin role
