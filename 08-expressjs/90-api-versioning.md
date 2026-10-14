# Lesson 90 — API Versioning

## Why API versioning matters

APIs evolve.

At some point you may need to make a breaking change.

Examples:

- rename fields
- change response shape
- change validation rules
- remove endpoints
- change business semantics

Versioning lets old clients continue working while new clients adopt the new contract.

---

## 1. What is a breaking change?

Suppose v1 returns:

```json
{
  "name": "Vikash"
}
```

v2 returns:

```json
{
  "firstName": "Vikash",
  "lastName": "Kumar"
}
```

Clients expecting `name` may break.

That is a breaking change.

---

## 2. URL versioning

Most common approach:

```text
/api/v1/users
/api/v2/users
```

Express:

```js
app.use(
  "/api/v1",
  v1Router
);

app.use(
  "/api/v2",
  v2Router
);
```

---

## 3. Advantages of URL versioning

- obvious
- easy to debug
- easy to document
- easy for clients
- easy to route

This is why it is widely used.

---

## 4. Header versioning

Example concept:

```http
Accept: application/vnd.myapp.v2+json
```

Advantages:
- URLs stay clean

Disadvantages:
- harder to discover
- harder to test manually
- more complex documentation

---

## 5. Custom header versioning

Example:

```http
X-API-Version: 2
```

Simple, but less standardized.

---

## 6. Query versioning

Example:

```text
/users?version=2
```

Usually less preferred for major API contracts because query params are generally better for resource retrieval options than protocol version identity.

---

## 7. Do not version every change

Non-breaking changes often do not require a new version.

Examples:

- adding optional response fields
- adding new endpoints
- adding optional query parameters

Versioning should be reserved for compatibility boundaries.

---

## 8. Version router architecture

```text
routes/
├── v1/
│   ├── user.routes.js
│   └── order.routes.js
└── v2/
    ├── user.routes.js
    └── order.routes.js
```

But duplicating everything can become expensive.

---

## 9. Share business logic

Do not copy entire services for every API version if business rules are unchanged.

Better:

```text
v1 controller
      |
      v
shared service

v2 controller
      |
      v
shared service
```

Only the API contract layer may differ.

---

## 10. Version transformation layer

Example:

Service returns:

```js
{
  firstName:
    "Vikash",
  lastName:
    "Kumar",
}
```

v1 controller transforms:

```js
{
  name:
    "Vikash Kumar"
}
```

v2 returns newer shape.

This avoids duplicating business logic.

---

## 11. Deprecation

Version lifecycle:

```text
v1 active
   |
   v
v2 released
   |
   v
v1 deprecated
   |
   v
migration window
   |
   v
v1 removed
```

Communicate deprecation clearly.

---

## 12. Deprecation headers

APIs can communicate lifecycle metadata using response headers where appropriate.

Examples may include:
- deprecation indicators
- sunset dates
- documentation links

Exact strategy depends on your API standards.

---

## 13. Backward compatibility

Before creating v2, ask:

> Can this change be made backward-compatible?

Example:

Instead of removing:

```json
{
  "name": "Vikash"
}
```

you may temporarily add:

```json
{
  "name": "Vikash",
  "firstName": "Vikash"
}
```

Then migrate clients gradually.

---

## 14. Internal vs public APIs

Public APIs usually require stronger versioning discipline because external consumers cannot be upgraded instantly.

Internal APIs may use:
- coordinated deployments
- contract testing
- shorter migration windows

Still, breaking changes should be controlled.

---

## 15. Database versioning is different

API version:

```text
/client contract
```

Database migration version:

```text
/schema evolution
```

Do not confuse them.

---

## 16. Common mistakes

### Mistake 1
Creating v2 for every small change.

### Mistake 2
Duplicating all business logic between versions.

### Mistake 3
Removing v1 immediately.

### Mistake 4
No deprecation plan.

### Mistake 5
Changing response shape silently.

---

## 17. Interview questions

### Why version an API?

To introduce breaking changes without immediately breaking existing clients.

### Most common versioning strategy?

URL versioning such as `/api/v1`.

### Should every new field require v2?

Usually no, if the change is backward-compatible.

### Where should version differences live?

Preferably in API contract/controllers/serializers, while sharing business logic where possible.

---

## 18. Strong interview answer

> API versioning is used when a breaking contract change cannot be made backward-compatible. The most common approach is URL versioning such as /api/v1 and /api/v2 because it is explicit and easy to document. I avoid duplicating business logic across versions; instead, I keep services shared and version the controller or response-mapping layer. I also plan deprecation and migration rather than removing older versions abruptly.

---

## Interview-Ready Summary

```text
API Versioning
   |
   +--> breaking changes
   +--> backward compatibility
   +--> /api/v1
   +--> /api/v2
   +--> deprecation
   +--> shared services

Version contract,
not every tiny change.
```

## Practice Task

Design v1 and v2 for a user endpoint where v1 returns:

```json
{
  "name": "Vikash Kumar"
}
```

and v2 returns separate first/last names.

Share the same service layer.


---

## Complete Working Example — API V1 and V2 in Express.js

This example shows **why** versioning is needed, **where** to create version-specific code, **how Express matches requests**, and **how to reuse one service without duplicating business logic**.

### 1. Understand the real problem first

Suppose an old React client calls:

```http
GET /api/users/1
```

and expects:

```json
{
  "id": 1,
  "name": "Vikash Kumar",
  "email": "vikash@gmail.com"
}
```

Its UI reads `user.name`. Later, your new frontend needs a different response:

```json
{
  "id": 1,
  "profile": {
    "firstName": "Vikash",
    "lastName": "Kumar"
  },
  "email": "vikash@gmail.com"
}
```

If you silently replace the old response, existing clients break because `user.name` is missing.

**Solution:** Keep `GET /api/v1/users/1` for old clients and introduce `GET /api/v2/users/1` for new clients. Both can read from the same database/service.

### 2. Folder structure

```text
express-practice/
├── package.json
├── server.js
└── src/
    ├── app.js
    ├── services/
    │   └── user.service.js
    ├── controllers/
    │   ├── v1/
    │   │   └── user.controller.js
    │   └── v2/
    │       └── user.controller.js
    └── routes/
        ├── v1/
        │   └── user.routes.js
        └── v2/
            └── user.routes.js
```

Install Express:

```bash
npm install express
```

Set `"type": "module"` in your `package.json` to use `import` syntax.

### 3. Build ONE shared service

`src/services/user.service.js`

```js
const users = [
  {
    id: 1,
    firstName: "Vikash",
    lastName: "Kumar",
    email: "vikash@gmail.com",
  },
];

export const getUserById = (id) => {
  return users.find((user) => user.id === Number(id));
};
```

The service returns the same internal user object for both API versions. It does not need to know whether the request came from V1 or V2.

### 4. V1 controller — preserve the old contract

`src/controllers/v1/user.controller.js`

```js
import { getUserById } from "../../services/user.service.js";

export const getUser = (req, res) => {
  const user = getUserById(req.params.id);

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  return res.json({
    id: user.id,
    name: `${user.firstName} ${user.lastName}`,
    email: user.email,
  });
};
```

Old frontend code using `user.name` continues to work.

### 5. V2 controller — return the new contract

`src/controllers/v2/user.controller.js`

```js
import { getUserById } from "../../services/user.service.js";

export const getUser = (req, res) => {
  const user = getUserById(req.params.id);

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  return res.json({
    id: user.id,
    profile: {
      firstName: user.firstName,
      lastName: user.lastName,
    },
    email: user.email,
  });
};
```

**What changes between versions?** Only the API response format. The underlying business data still comes from `getUserById()`.

### 6. Version-specific routers

`src/routes/v1/user.routes.js`

```js
import { Router } from "express";
import { getUser } from "../../controllers/v1/user.controller.js";

const router = Router();
router.get("/:id", getUser);

export default router;
```

`src/routes/v2/user.routes.js`

```js
import { Router } from "express";
import { getUser } from "../../controllers/v2/user.controller.js";

const router = Router();
router.get("/:id", getUser);

export default router;
```

Both routers use `/:id`, but each points to the appropriate controller.

### 7. Register version prefixes with app.use()

`src/app.js`

```js
import express from "express";
import v1UserRouter from "./routes/v1/user.routes.js";
import v2UserRouter from "./routes/v2/user.routes.js";

const app = express();
app.use(express.json());

app.use("/api/v1/users", v1UserRouter);
app.use("/api/v2/users", v2UserRouter);

export default app;
```

`server.js`

```js
import app from "./src/app.js";

app.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});
```

Express URL mounting is straightforward:

```text
app.use("/api/v1/users", v1UserRouter)
                      +
router.get("/:id", ...)
                      =
GET /api/v1/users/1
```

### 8. Visualize the complete request flow

```text
                  Client request
                        |
                        v
                  Express app.js
                        |
          +-------------+-------------+
          |                           |
 GET /api/v1/users/1          GET /api/v2/users/1
          |                           |
          v                           v
     V1 user router               V2 user router
          |                           |
          v                           v
     V1 controller                V2 controller
          |                           |
          +------------+--------------+
                       |
                       v
                Shared user service
                       |
                       v
             Same user object / database
                       |
          +------------+------------+
          |                         |
          v                         v
    V1 transforms result       V2 transforms result
          |                         |
          v                         v
    { "name": "..." }         { "profile": {...} }
```

**Key idea:** Different URL versions do not require separate servers, services, or databases. Keep shared logic shared unless the business behavior itself has changed.

### 9. Run and test both URLs

```bash
node server.js

curl http://localhost:3000/api/v1/users/1
curl http://localhost:3000/api/v2/users/1
```

Expected V1:

```json
{
  "id": 1,
  "name": "Vikash Kumar",
  "email": "vikash@gmail.com"
}
```

Expected V2:

```json
{
  "id": 1,
  "profile": {
    "firstName": "Vikash",
    "lastName": "Kumar"
  },
  "email": "vikash@gmail.com"
}
```

`GET /api/v1/users/99` or `GET /api/v2/users/99` will return `404` with `{ "message": "User not found" }` in this simplified example. In a larger application, you can reuse the centralized error handler from Lesson 87.

### 10. When do you actually need V2?

| Change | Usually requires V2? | Why |
|---|---|---|
| Fix a bug internally | No | Client contract stays the same |
| Improve SQL query speed | No | Invisible to clients |
| Add an optional response field | No | Usually compatible |
| Add a new endpoint | No | Existing endpoints keep working |
| Rename `name` to `profile` | Yes | Old clients break |
| Remove required response fields | Yes | Old clients may depend on them |
| Change existing field types | Yes | Old clients may break |
| Make previously optional input mandatory | Yes | Old requests may fail |

The real question is: **Will existing clients continue working without code changes?** If yes, you may not need a new major API version.

### 11. Other API versioning approaches

- **URL versioning:** `/api/v1/users` — explicit and easy to debug (recommended for this exercise).
- **Custom header:** `API-Version: 2` — one URL, but version is less visible.
- **Media type / Accept header:** `Accept: application/vnd.myapp.v2+json` — versioned representation.
- **Query parameter:** `/api/users?version=2` — simple, but usually less preferred for major contracts.

Express itself does not require a special versioning library. Versioned URL prefixes and `Router()` are enough for URL-based versioning.

### 12. What happens when V3 arrives? Deprecation lifecycle

```text
V1 active
   |
   v
V2 released alongside V1
   |
   v
Clients migrate to V2
   |
   v
V1 deprecated (documentation + notice)
   |
   v
Migration window + monitor remaining V1 traffic
   |
   v
V1 removed after communicated sunset date
```

Don't delete V1 immediately after releasing V2. Existing browsers, mobile apps, and external integrations may still depend on V1.

Also remember: **API versioning is not application versioning or database migration versioning.** You may deploy backend code many times without changing the API contract.

### 13. Interview-ready takeaway

> API versioning lets us introduce backward-incompatible request or response changes without breaking existing consumers. In Express, I can mount separate routers at `/api/v1` and `/api/v2`, keep the controllers or serializers version-specific, and reuse shared services and databases. I introduce new major API versions for breaking changes and use a planned deprecation period to migrate clients.

**Remember in one line:** Version the **public contract**, not every small code change.

