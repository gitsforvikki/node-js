# Lesson 206 — Production Node.js Project Structure

## 1. Why does structure matter?
A tiny Express project can fit into `server.js`. As authentication, orders, jobs and tests grow, one file becomes difficult to maintain. Structure helps developers find code, isolate responsibilities and test without starting the whole server.

## 2. Realistic Express 5 + Mongoose layout
```text
src/
  app.js                 # Express configuration, middlewares, routes
  server.js              # Database connection, listen, shutdown
  config/env.js          # Validate environment config
  db/mongoose.js         # Database setup
  modules/
    users/
      user.routes.js
      user.controller.js
      user.service.js
      user.repository.js
      user.model.js
      user.validation.js
    orders/
      order.routes.js
      order.controller.js
      order.service.js
  middlewares/
    requireAuth.js
    errorHandler.js
  errors/AppError.js
  lib/logger.js
tests/
  unit/
  integration/
Dockerfile
.dockerignore
.env.example
package.json
```

**Rule:** Organize by feature when features grow; share only truly reusable infrastructure.

## 3. Separate app creation and starting server
**src/app.js**
```js
import express from "express";
import { userRouter } from "./modules/users/user.routes.js";

export function createApp() {
  const app = express();
  app.use(express.json({ limit: "16kb" }));
  app.use("/api/users", userRouter);
  app.use((err, req, res, next) => {
    if (res.headersSent) return next(err);
    console.error(err);
    res.status(500).json({ message: "Internal server error" });
  });
  return app;
}
```

**src/server.js**
```js
import mongoose from "mongoose";
import { createApp } from "./app.js";

async function start() {
  if (!process.env.MONGODB_URI) throw new Error("MONGODB_URI is required");
  await mongoose.connect(process.env.MONGODB_URI);
  createApp().listen(process.env.PORT || 3000);
}
start().catch((err) => {
  console.error("Startup failed:", err.message);
  process.exitCode = 1;
});
```

**Why:** integration tests can import `createApp()` without connecting to production databases or opening a permanent HTTP port. Production startup happens only in `server.js`.

## 4. Example request flow
```text
GET /api/users/123
  -> route (defines endpoint)
  -> middleware (auth and validation)
  -> controller (HTTP request/response)
  -> service (business rules)
  -> repository (Mongoose query)
  -> MongoDB
```

## 5. Architecture decisions
Small MVP: route + controller + model can be enough. Add services when business rules grow; repositories when persistence details repeat or need isolation. **Don't create empty layers just because the folder diagram shows them.**

## 6. Production essentials
Add config validation, error handling, request IDs, structured logging, health checks, tests, graceful shutdown, secure dependencies and environment-specific configuration. Keep secrets out of Git; use `.env.example` for variable names only.

## 7. Common mistakes
Mixing `app.listen()` inside `app.js`; controllers directly sending emails and updating many models; circular imports; giant `utils.js`; importing configuration before loading env variables.

## Practice
Refactor a single-file CRUD API into `app.js`, `server.js`, `users/` module and an integration test using Supertest.

**Interview answer:** Production structure separates HTTP setup, startup, feature modules, business logic, persistence and shared cross-cutting concerns. Structure should support testability and changes without unnecessary abstraction.
