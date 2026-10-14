# Lesson 78 — Express Application Setup

## Why setup quality matters

A good Express app should not start as one huge `server.js` file.

Even small projects benefit from separating:

- app configuration
- server startup
- routes
- controllers
- middleware
- config

This makes testing and deployment easier.

---

## 1. Install Express

```bash
npm install express
```

With ES Modules:

```json
{
  "type": "module"
}
```

---

## 2. Minimal app

```js
import express from "express";

const app =
  express();

app.get(
  "/",
  (req, res) => {
    res.send("Hello Express");
  }
);

app.listen(
  3000,
  () => {
    console.log(
      "Server running"
    );
  }
);
```

This works.

But production structure can be better.

---

## 3. Separate app and server

Recommended pattern:

```text
src/
├── app.js
├── server.js
└── routes/
```

### app.js

```js
import express from "express";

export const app =
  express();

app.use(
  express.json()
);
```

### server.js

```js
import { app }
  from "./app.js";

const PORT =
  Number(
    process.env.PORT ?? 3000
  );

app.listen(
  PORT,
  () => {
    console.log(
      `Server running on ${PORT}`
    );
  }
);
```

---

## 4. Why split app and server?

Because tests can import:

```js
import { app }
  from "../src/app.js";
```

without automatically opening a network port.

This improves integration testing.

---

## 5. Add built-in body middleware

```js
app.use(
  express.json()
);
```

For URL-encoded forms:

```js
app.use(
  express.urlencoded({
    extended: true,
  })
);
```

Use only what your application needs.

---

## 6. Body size limit

Do not accept unlimited JSON.

Example:

```js
app.use(
  express.json({
    limit: "1mb",
  })
);
```

This protects memory.

---

## 7. Basic route setup

```js
app.get(
  "/health",
  (req, res) => {
    res.status(200).json({
      status: "ok",
    });
  }
);
```

---

## 8. Mount routers

```js
import userRouter
  from "./routes/user.routes.js";

app.use(
  "/api/users",
  userRouter
);
```

This keeps app.js small.

---

## 9. Common production structure

```text
src/
├── app.js
├── server.js
├── config/
│   └── env.js
├── routes/
│   └── user.routes.js
├── controllers/
│   └── user.controller.js
├── services/
│   └── user.service.js
├── repositories/
│   └── user.repository.js
├── middlewares/
│   ├── auth.middleware.js
│   └── error.middleware.js
└── utils/
```

### How routes, controllers, services, and repositories work together

Consider a **User Management API** that accepts `POST /api/users`:

```json
{
  "name": "Vikash",
  "email": "vikash@gmail.com"
}
```

Each layer has a separate responsibility:

| Layer | Main responsibility | Example |
|---|---|---|
| Route | Choose which controller handles an HTTP method and URL | `router.post("/", createUser)` |
| Controller | Read the HTTP request, call a service, and send the response | `req.body`, `res.status(201).json(...)` |
| Service | Apply business rules and coordinate operations | Require name/email; reject existing email |
| Repository | Run database operations | `User.findOne()`, `User.create()` |

```text
Client: POST /api/users
         |
         v
app.js: mounts /api/users
         |
         v
user.routes.js: POST "/" -> createUser
         |
         v
user.controller.js: read req.body, call service
         |
         v
user.service.js: validate input, check duplicate
         |
         v
user.repository.js: query / insert database
         |
         v
Database
```

The result flows back **repository → service → controller → client**. The route selects the controller; it does not run the business logic itself.

#### Step 1 — `routes/user.routes.js`

```js
import { Router } from "express";
import { createUser } from "../controllers/user.controller.js";

const router = Router();

router.post("/", createUser);

export default router;
```

The router maps `POST /` to the `createUser` controller. Because `app.js` mounts it under `/api/users`, the full endpoint is `POST /api/users`.

#### Step 2 — `controllers/user.controller.js`

```js
import { createUserService } from "../services/user.service.js";

export async function createUser(req, res, next) {
  try {
    const user = await createUserService(req.body);

    return res.status(201).json({
      success: true,
      message: "User created successfully",
      data: user,
    });
  } catch (error) {
    next(error);
  }
}
```

The controller knows about **HTTP** (`req`, `res`, status `201`), but delegates business rules and database access to other layers. `next(error)` forwards errors to the error middleware (Section 12).

#### Step 3 — `services/user.service.js`

```js
import {
  findUserByEmail,
  saveUser,
} from "../repositories/user.repository.js";

export async function createUserService(data) {
  const name = typeof data?.name === "string" ? data.name.trim() : "";
  const email = typeof data?.email === "string"
    ? data.email.trim().toLowerCase()
    : "";

  if (!name || !email) {
    const error = new Error("Name and email are required");
    error.statusCode = 400;
    throw error;
  }

  const existingUser = await findUserByEmail(email);

  if (existingUser) {
    const error = new Error("Email already registered");
    error.statusCode = 409;
    throw error;
  }

  return saveUser({ name, email });
}
```

The service handles **business rules** without depending on Express `req` or `res`. This makes the same service reusable from a CLI command, another controller, or a background job. A real application should also validate email format and enforce a **unique database index** to avoid duplicate users under concurrent requests.

#### Step 4 — `repositories/user.repository.js`

This example uses Mongoose (assume the `User` model is defined in `models/user.model.js` and the database connection is initialized during startup):

```js
import User from "../models/user.model.js";

export function findUserByEmail(email) {
  return User.findOne({ email });
}

export function saveUser(data) {
  return User.create(data);
}
```

The repository owns **database access**, not HTTP responses or business decisions.

#### Step 5 — Connect the router in `app.js`

```js
import express from "express";
import userRoutes from "./routes/user.routes.js";

export const app = express();

app.use(express.json({ limit: "1mb" }));
app.use("/api/users", userRoutes);

// Register 404 and error middleware after the routes.
```

`server.js` imports `app` and starts listening as shown in Section 3; for database-backed routes, connect the database **before** `listen()` (Section 14).

#### Why not put everything in the controller?

You *could* make database queries directly in an Express route, but the same validation and duplicate-checking rules may then get repeated across API routes, admin features, and background jobs. Separation makes these rules easier to reuse and unit-test.

**Memory trick:**

- **Route:** Where does the request go?
- **Controller:** What HTTP input arrived, and what HTTP response goes out?
- **Service:** What rules must the application follow?
- **Repository:** How does it read/write data?

**Interview note:** Not every small CRUD app needs four layers. Routes + controllers + services may be sufficient; add repositories when separating persistence logic meaningfully improves the codebase.

**Production note:** The illustrative service throws errors with `statusCode`. The generic error handler in Section 12 always returns 500; update that handler to honor validated operational status codes (such as 400 and 409) when implementing this example fully.

---

## 10. Environment config

```js
const PORT =
  Number(
    process.env.PORT ?? 3000
  );
```

Do not hardcode production config everywhere.

Centralize it.

---

## 11. 404 handler

After all routes:

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

Order matters.

---

## 12. Error middleware

Express error middleware uses four parameters:

```js
app.use(
  (
    err,
    req,
    res,
    next
  ) => {
    console.error(err);

    res
      .status(500)
      .json({
        message:
          "Internal server error",
      });
  }
);
```

That four-argument shape is important.

---

## 13. Middleware order

Express executes middleware in registration order.

Example:

```js
app.use(logger);
app.use(express.json());
app.use("/api", apiRouter);
app.use(notFoundHandler);
app.use(errorHandler);
```

Conceptually:

```text
Request
   |
   v
logger
   |
   v
body parser
   |
   v
routes
   |
   v
404
   |
   v
error middleware if error
```

---

## 14. Async server startup

Often you want to initialize dependencies before listening.

```js
async function start() {
  await connectDatabase();

  app.listen(
    PORT,
    () => {
      console.log("ready");
    }
  );
}

start().catch(
  (error) => {
    console.error(error);
    process.exit(1);
  }
);
```

This avoids accepting traffic before dependencies are ready.

---

## 15. Graceful shutdown

Production apps should handle signals:

```js
const server =
  app.listen(PORT);

process.on(
  "SIGTERM",
  async () => {
    server.close(
      async () => {
        await closeDatabase();

        process.exit(0);
      }
    );
  }
);
```

---

## 16. Trust proxy

Behind a reverse proxy/load balancer, settings such as client IP or secure cookies may depend on proxy trust configuration.

Example:

```js
app.set(
  "trust proxy",
  1
);
```

Only configure this according to your actual infrastructure.

Do not blindly enable proxy trust.

---

## 17. Security basics

Typical production setup may later include:

- Helmet
- CORS configuration
- rate limiting
- secure cookies
- request logging
- validation

These are not automatically enabled by Express.

---

## 18. Common mistakes

### Mistake 1
Putting all routes/controllers in one file.

### Mistake 2
Starting server before DB connection succeeds.

### Mistake 3
No body limits.

### Mistake 4
Wrong middleware order.

### Mistake 5
No centralized error handler.

### Mistake 6
No graceful shutdown.

---

## 19. Interview questions

### Why separate app.js and server.js?

It improves testing and keeps app configuration separate from network startup.

### Why does middleware order matter?

Express processes middleware in registration order.

### Why should dependencies initialize before listen?

To avoid accepting traffic before the service is actually ready.

### Why use express.json with a limit?

To parse JSON safely while controlling memory usage.

---

## 20. Strong interview answer

> I usually separate Express application configuration from server startup. app.js registers middleware, routes, 404 handling, and error middleware, while server.js loads configuration, connects external dependencies, starts listening, and handles graceful shutdown. This structure improves testability and prevents the service from accepting traffic before it is ready.

---

## Interview-Ready Summary

```text
app.js
  -> middleware
  -> routers
  -> 404
  -> errors

server.js
  -> config
  -> DB connection
  -> listen()
  -> graceful shutdown

Important:
middleware order matters
```

## Practice Task

Create a clean Express starter with:

- app.js
- server.js
- /health route
- JSON body limit
- 404 handler
- centralized error handler
- graceful shutdown
