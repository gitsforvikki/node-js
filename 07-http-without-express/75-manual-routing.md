# Lesson 75 — Building Routing Manually

## Why this lesson matters

Routing frameworks feel simple because they hide a real problem:

> Given an HTTP method and URL, which handler should run?

Building a small router manually helps you understand:
- method matching
- static routes
- dynamic params
- query parsing
- 404 vs 405
- handler architecture

---

## 1. Simplest router

```js
if (
  req.method === "GET" &&
  req.url === "/users"
) {
  // handler
}
```

This works for tiny apps.

It does not scale well.

---

## 2. Parse pathname first

Always separate pathname from query string.

```js
const url =
  new URL(
    req.url,
    `http://${req.headers.host}`
  );

const pathname =
  url.pathname;
```

Otherwise:

```text
/users?page=2
```

will not equal:

```text
/users
```

---

## 3. Route table

A cleaner design:

```js
const routes = [
  {
    method: "GET",
    path: "/users",
    handler: listUsers,
  },

  {
    method: "POST",
    path: "/users",
    handler: createUser,
  },
];
```

Then match:

```js
const route =
  routes.find(
    (route) =>
      route.method ===
        req.method &&
      route.path ===
        pathname
  );
```

---

## 4. Static vs dynamic routes

Static:

```text
/users
/health
/orders
```

Dynamic:

```text
/users/:id
/orders/:orderId
```

Your router must detect both.

---

## 5. Manual pattern matcher

Concept:

```text
Pattern:
/users/:id

Actual:
/users/123

Segments:

users == users
:id   -> capture 123
```

---

## 6. Simple matcher

```js
function matchRoute(
  pattern,
  pathname
) {
  const patternParts =
    pattern
      .split("/")
      .filter(Boolean);

  const pathParts =
    pathname
      .split("/")
      .filter(Boolean);

  if (
    patternParts.length !==
    pathParts.length
  ) {
    return null;
  }

  const params = {};

  for (
    let i = 0;
    i <
    patternParts.length;
    i++
  ) {
    const patternPart =
      patternParts[i];

    const pathPart =
      pathParts[i];

    if (
      patternPart.startsWith(
        ":"
      )
    ) {
      params[
        patternPart.slice(1)
      ] =
        decodeURIComponent(
          pathPart
        );

      continue;
    }

    if (
      patternPart !==
      pathPart
    ) {
      return null;
    }
  }

  return params;
}
```

---

## 7. Router flow

```text
Request
   |
   v
parse URL
   |
   v
find matching pathname
   |
   v
check method
   |
   v
extract params
   |
   v
run handler
```

---

## 8. 404 vs 405

This is important.

If no path matches:

```text
404 Not Found
```

If path exists but method does not:

```text
405 Method Not Allowed
```

Example:

```text
Route exists:
GET /users

Request:
DELETE /users

Result:
405
```

---

## 9. Middleware concept

You can manually create middleware-like functions.

Example:

```js
async function authenticate(
  req
) {
  // verify token
}
```

Then:

```js
await authenticate(req);

await handler(
  req,
  res
);
```

This shows where Express middleware architecture comes from.

---

## 10. Context object

Instead of mutating raw objects heavily, build route context:

```js
const context = {
  req,
  res,
  params,
  query:
    Object.fromEntries(
      url.searchParams
    ),
};
```

This makes handler interfaces clearer.

---

## 11. Handler separation

Bad:

```js
http.createServer(
  (req, res) => {
    // 500 lines
  }
);
```

Better:

```text
server.js
router.js
handlers/
  users.js
  orders.js
```

The HTTP server should mostly delegate.

### Example — User and Order API without Express

Suppose we need these endpoints:

| Method | URL | Purpose |
|---|---|---|
| GET | `/users` | List users |
| POST | `/users` | Create a user |
| GET | `/orders` | List orders |
| POST | `/orders` | Create an order |

Instead of putting every route in `createServer()`, give each file one responsibility:

- **server.js:** starts the HTTP server.
- **router.js:** checks the pathname and method, then selects a handler.
- **handlers/users.js:** implements user operations.
- **handlers/orders.js:** implements order operations.

#### Step 1 — `handlers/users.js`

```js
const users = [{ id: 1, name: "Vikash" }];

export function getUsers(req, res) {
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify(users));
}

export function createUser(req, res) {
  let body = "";

  req.on("data", (chunk) => {
    body += chunk.toString();
  });

  req.on("end", () => {
    try {
      const data = JSON.parse(body);
      if (typeof data.name !== "string" || !data.name.trim()) {
        res.writeHead(400);
        return res.end("A name is required");
      }

      const user = {
        id: users.length + 1,
        name: data.name.trim(),
      };
      users.push(user);

      res.writeHead(201, { "Content-Type": "application/json" });
      res.end(JSON.stringify(user));
    } catch {
      res.writeHead(400);
      res.end("Invalid JSON");
    }
  });
}
```

#### Step 2 — `handlers/orders.js`

```js
const orders = [{ id: 1, product: "Laptop" }];

export function getOrders(req, res) {
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify(orders));
}

export function createOrder(req, res) {
  let body = "";

  req.on("data", (chunk) => {
    body += chunk.toString();
  });

  req.on("end", () => {
    try {
      const data = JSON.parse(body);
      if (typeof data.product !== "string" || !data.product.trim()) {
        res.writeHead(400);
        return res.end("A product is required");
      }

      const order = {
        id: orders.length + 1,
        product: data.product.trim(),
      };
      orders.push(order);

      res.writeHead(201, { "Content-Type": "application/json" });
      res.end(JSON.stringify(order));
    } catch {
      res.writeHead(400);
      res.end("Invalid JSON");
    }
  });
}
```

#### Step 3 — `router.js`

The router **chooses** a handler. It does not implement user or order logic.

```js
import { getUsers, createUser } from "./handlers/users.js";
import { getOrders, createOrder } from "./handlers/orders.js";

export function router(req, res) {
  const { pathname } = new URL(req.url, "http://localhost");

  const routes = {
    "/users": { GET: getUsers, POST: createUser },
    "/orders": { GET: getOrders, POST: createOrder },
  };

  const methods = routes[pathname];

  if (!methods) {
    res.writeHead(404);
    return res.end("Route not found");
  }

  const handler = methods[req.method];

  if (!handler) {
    res.writeHead(405, { Allow: Object.keys(methods).join(", ") });
    return res.end("Method not allowed");
  }

  return handler(req, res);
}
```

Notice the difference: **404** means the path is unknown; **405** means the path exists but does not support the HTTP method (see Section 8).

#### Step 4 — `server.js`

```js
import http from "node:http";
import { router } from "./router.js";

const server = http.createServer(router);

server.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});
```

For ES module imports, use this `package.json`:

```json
{
  "type": "module",
  "scripts": {
    "dev": "node --watch server.js"
  }
}
```

Run with `npm run dev`.

#### Step 5 — Test it

```bash
curl http://localhost:3000/users

curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Rahul"}'

curl http://localhost:3000/orders

curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{"product":"Mouse"}'
```

**Request flow:**

```text
Client sends GET /users
        |
        v
server.js receives request
        |
        v
router.js matches GET + /users
        |
        v
handlers/users.js -> getUsers()
        |
        v
JSON response goes to client
```

**Why this helps:** as more endpoints are added, the server stays small and each handler can be read and tested independently. This is the same separation of concerns used by frameworks such as Express.

**Learning-example limitation:** data is stored in arrays and disappears when the server restarts. A production server also needs request-body size limits, robust stream/error handling, persistent storage, and centralized error handling (Section 12). The example focuses on **handler separation** rather than building a complete production router.


---

## 12. Error boundary

Wrap async route handling:

```js
try {
  await route.handler(
    context
  );
} catch (error) {
  handleError(
    error,
    res
  );
}
```

Without a central boundary, every handler repeats error logic.

---

## 13. Route ordering

Dynamic routes can conflict with static routes.

Example:

```text
/users/search
/users/:id
```

If dynamic matching is naive:

```text
search
```

may accidentally become:

```text
id = "search"
```

Framework routers solve this using route precedence/matching logic.

---

## 14. Normalize paths carefully

Potential differences:

```text
/users
/users/
/users//123
```

Decide how your router treats trailing slashes and malformed paths.

---

## 15. Security concerns

Manual routers must still handle:
- decoding failures
- malformed paths
- path traversal where filesystem paths are involved
- request-size limits
- authentication
- authorization

Routing does not equal security.

---

## 16. Why frameworks exist

After building this manually, Express becomes easier to appreciate.

Express gives you:

```js
app.get(
  "/users/:id",
  handler
);
```

instead of manually implementing:
- route table
- matcher
- param extraction
- middleware chain
- error flow

---

## 17. Interview questions

### What does a router do?

Matches an HTTP method and URL pattern to a handler, often extracting parameters and applying middleware.

### Difference between 404 and 405?

404 means no resource/route matches; 405 means the path exists but the HTTP method is unsupported.

### Why parse pathname separately from query string?

Because routing should generally match the path independently from optional query parameters.

---

## 18. Strong interview answer

> A router maps a request's method and pathname to a handler. In raw Node.js I first parse the URL, then match static or dynamic route patterns, extract route parameters, distinguish 404 from 405, and call the selected handler inside a centralized error boundary. Frameworks such as Express automate this routing and middleware pipeline.

---

## Interview-Ready Summary

```text
Router
   |
   +--> method
   +--> pathname
   +--> dynamic params
   +--> query
   +--> middleware
   +--> handler
   +--> errors

404:
no route

405:
path exists, method invalid
```

## Practice Task

Build a tiny router supporting:

```text
GET    /users
POST   /users
GET    /users/:id
PATCH  /users/:id
DELETE /users/:id
```

Requirements:
- dynamic params
- query parsing
- 404
- 405
- centralized error handling
