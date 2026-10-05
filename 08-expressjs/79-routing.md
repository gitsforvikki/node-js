# Lesson 79 — Routing in Express

## Why routing matters

Routing maps:

```text
HTTP Method + URL
        |
        v
     Handler
```

Express makes this much simpler than manual routing.

---

## 1. Basic routes

```js
app.get(
  "/users",
  getUsers
);

app.post(
  "/users",
  createUser
);

app.patch(
  "/users/:id",
  updateUser
);

app.delete(
  "/users/:id",
  deleteUser
);
```

---

## 2. Route methods map to HTTP methods

```text
app.get     -> GET
app.post    -> POST
app.put     -> PUT
app.patch   -> PATCH
app.delete  -> DELETE
```

This makes API intent explicit.

---

## 3. Route handler

```js
function getUsers(
  req,
  res
) {
  res.json([]);
}
```

Express calls the handler when the route matches.

---

## 4. Multiple handlers

A route can have multiple middleware/handlers:

```js
app.get(
  "/profile",
  authenticate,
  authorizeUser,
  getProfile
);
```

Flow:

```text
/profile
   |
   v
authenticate
   |
   v
authorizeUser
   |
   v
getProfile
```

---

## 5. express.Router()

For modular routing:

```js
import {
  Router,
} from "express";

const router =
  Router();

router.get(
  "/",
  listUsers
);

router.post(
  "/",
  createUser
);

export default router;
```

Mount:

```js
app.use(
  "/api/users",
  router
);
```

---

## 6. Mounted path composition

If router has:

```js
router.get(
  "/:id",
  getUser
);
```

and app mounts:

```js
app.use(
  "/api/users",
  router
);
```

final route is:

```text
GET /api/users/:id
```

---

## 7. Resource-based route files

Good structure:

```text
routes/
├── user.routes.js
├── order.routes.js
└── auth.routes.js
```

This scales better than one route file.

---

## 8. Controller separation

Route:

```js
router.get(
  "/:id",
  getUser
);
```

Controller:

```js
export async function getUser(
  req,
  res,
  next
) {
  // request/response logic
}
```

Service:

```js
await userService.getById(
  req.params.id
);
```

This separation keeps routing declarative.

---

## 9. Route order matters

Example:

```js
router.get(
  "/:id",
  getUser
);

router.get(
  "/search",
  searchUsers
);
```

Depending on route matching behavior, dynamic routes can capture static segments unexpectedly.

Safer:

```js
router.get(
  "/search",
  searchUsers
);

router.get(
  "/:id",
  getUser
);
```

Specific routes should usually come before broad dynamic ones.

---

## 10. app.use vs app.METHOD

```js
app.use(
  "/api",
  middleware
);
```

matches multiple HTTP methods under that path.

```js
app.get(
  "/api",
  handler
);
```

matches only GET.

---

## 11. Route-level middleware

```js
router.post(
  "/",
  authenticate,
  validateCreateUser,
  createUser
);
```

This gives route-specific behavior.

---

## 12. Router-level middleware

```js
router.use(
  authenticate
);
```

Everything registered after this can inherit the middleware.

Useful for protected route groups.

---

## 13. 404 handling

If no route matches:

```js
app.use(
  (req, res) => {
    res
      .status(404)
      .json({
        message:
          "Route not found",
      });
  }
);
```

Place after routes.

---

## 14. 405 handling

Express does not automatically give perfect 405 handling for every design.

If you need strict method reporting, you may implement method checks or route-specific fallback handlers.

---

## 15. RESTful routing

Typical resource routes:

```text
GET    /users
GET    /users/:id
POST   /users
PATCH  /users/:id
DELETE /users/:id
```

Avoid:

```text
POST /getUsers
POST /deleteUser
```

unless you intentionally choose RPC-style semantics.

---

## 16. Common mistakes

### Mistake 1
Putting business logic directly in route files.

### Mistake 2
Bad route order.

### Mistake 3
Using app.use when method-specific route is needed.

### Mistake 4
Huge routers with unrelated resources.

### Mistake 5
No route-level validation/auth middleware.

---

## 17. Interview questions

### What is express.Router?

A mini router that groups related routes and middleware.

### Why split routes into routers?

For modularity, separation, and cleaner mounting.

### Why can route order matter?

Broad dynamic patterns may match requests before more specific routes.

### Difference between app.use and app.get?

`app.use` mounts middleware for matching paths across methods; `app.get` handles GET requests only.

---

## 18. Strong interview answer

> Express routing maps HTTP methods and path patterns to handlers. For larger applications I use express.Router to group routes by resource, keep route files declarative, attach validation/auth middleware at route or router level, and delegate business logic to controllers/services. Route order matters when dynamic patterns overlap with static routes.

---

## Interview-Ready Summary

```text
Routing
   |
   +--> method
   +--> path
   +--> middleware
   +--> controller

express.Router
   -> modular route groups

Important:
route order
resource separation
keep business logic out of routes
```

## Practice Task

Create:

```text
/api/users
/api/orders
```

with separate routers and CRUD-style methods.
