# Lesson 70 — HTTP Methods

## Why HTTP methods matter

HTTP methods communicate the **intent** of a request.

Good API design uses methods consistently.

Main methods:

```text
GET
POST
PUT
PATCH
DELETE
HEAD
OPTIONS
```

---

## 1. GET

Used to retrieve a resource.

Example:

```http
GET /users/123
```

Expected behavior:
- read data
- should not create/update server state as its intended semantic effect

### Practical example — GET: Read all users

```js
const response = await fetch("http://localhost:3000/users");
console.log(await response.json());
```

GET requests read information without intentionally changing server data.

---

## 2. POST

Commonly used to create a resource or trigger a non-idempotent operation.

Example:

```http
POST /users
```

Body:

```json
{
  "name": "Vikash"
}
```

Possible response:

```text
201 Created
```

### Practical example — POST: Create a user

```js
const response = await fetch("http://localhost:3000/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Vikash", city: "Patna" }),
});
console.log(response.status); // 201
console.log(await response.json());
```

POST creates a new user. The raw Node.js server must read request-body chunks manually.

---

## 3. PUT

Typically means:

> replace the resource representation at this URI

Example:

```http
PUT /users/123
```

Body may contain the full desired representation.

### Practical example — PUT: Replace a user

```js
const response = await fetch("http://localhost:3000/users/1", {
  method: "PUT",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Vikash Kumar", city: "Pune" }),
});
console.log(await response.json());
```

PUT replaces the whole editable user representation. Include every field to retain.

---

## 4. PATCH

Used for partial modification.

Example:

```http
PATCH /users/123
```

Body:

```json
{
  "name": "New Name"
}
```

### Practical example — PATCH: Update a single field

```js
const response = await fetch("http://localhost:3000/users/1", {
  method: "PATCH",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ city: "Bengaluru" }),
});
console.log(await response.json());
```

PATCH modifies only the provided fields in this demo; the name stays unchanged.

---

## 5. DELETE

Used to remove a resource.

```http
DELETE /users/123
```

Possible responses:
- 204 No Content
- 200 with result body

### Practical example — DELETE: Remove a user

```js
const response = await fetch("http://localhost:3000/users/1", {
  method: "DELETE",
});
console.log(response.status); // 204 if deleted
// No response.json(): 204 responses have no body.
```

DELETE removes the user. A repeated request can return 404 even though DELETE is idempotent in its intended effect.

---

## 6. HEAD

HEAD is similar to GET but response body is omitted.

Useful for:
- metadata
- existence checks
- content length
- caching logic

### Practical example — HEAD: Read response headers

```js
const response = await fetch("http://localhost:3000/users", {
  method: "HEAD",
});
console.log(response.status); // 200
console.log(response.headers.get("content-type"));
```

HEAD is similar to GET but has no response body.

---

## 7. OPTIONS

Used to describe communication options for a resource/server.

Very common in CORS preflight requests.

Example:

```http
OPTIONS /users
```

### Practical example — OPTIONS: Read allowed methods

```js
const response = await fetch("http://localhost:3000/users", {
  method: "OPTIONS",
});
console.log(response.status); // 204
console.log(response.headers.get("allow"));
```

The Allow header describes supported methods. CORS preflight requires additional Access-Control-Allow-* headers.

---

# Safety and Idempotency

These terms are heavily tested.

---

## 8. Safe methods

A safe method is intended not to change server state.

Examples:

```text
GET
HEAD
OPTIONS
```

Safe does not mean:
- no logging
- no analytics
- no cache updates

It means the requested semantic action is read-only.

---

## 9. Idempotent methods

An operation is idempotent if repeating the same request has the same intended effect as doing it once.

Examples typically considered idempotent:

```text
GET
PUT
DELETE
HEAD
OPTIONS
```

POST is generally not assumed idempotent.

PATCH may or may not be idempotent depending on semantics.

---

## 10. Idempotency example

DELETE:

```http
DELETE /users/123
```

First call:
- user deleted

Second call:
- user already absent

The resulting resource state is still:

```text
user does not exist
```

That is why DELETE is idempotent semantically.

---

## 11. POST payment problem

Suppose:

```http
POST /payments
```

Client sends request.

Payment succeeds, but response is lost.

Client retries.

Now two payments may be created.

Solution for critical POST operations:

```text
Idempotency-Key
```

The server recognizes retries of the same logical operation.

---

## 12. PUT vs PATCH

```text
PUT
  -> complete replacement semantics

PATCH
  -> partial modification
```

Example user:

```json
{
  "name": "Vikash",
  "city": "Bengaluru"
}
```

PATCH:

```json
{
  "city": "Pune"
}
```

Only city changes.

---

## 13. Method handling in raw Node.js ⭐⭐⭐⭐⭐

The examples above use this **complete runnable server without Express**. Save as `server.mjs` and run `node server.mjs` (Node.js 18+). Users are stored in memory, so data resets when you restart the server.

```js
import http from "node:http";

// In-memory data: restart server to reset.
const users = new Map([[1, { id: 1, name: "Vikash", city: "Patna" }]]);
let nextId = 2;

function json(res, status, data) {
  res.writeHead(status, { "Content-Type": "application/json; charset=utf-8" });
  res.end(JSON.stringify(data));
}

async function readBody(req) {
  let body = "";
  for await (const chunk of req) body += chunk;
  return JSON.parse(body);
}

const server = http.createServer(async (req, res) => {
  const { pathname } = new URL(req.url, "http://localhost");
  const method = req.method;

  if (pathname === "/users") {
    if (method === "GET") return json(res, 200, [...users.values()]);
    if (method === "HEAD") {
      res.writeHead(200, { "Content-Type": "application/json; charset=utf-8" });
      return res.end();
    }
    if (method === "OPTIONS") {
      res.writeHead(204, { Allow: "GET, HEAD, POST, OPTIONS" });
      return res.end();
    }
    if (method === "POST") {
      try {
        const input = await readBody(req);
        if (!input || typeof input.name !== "string" ||
            typeof input.city !== "string") {
          return json(res, 400, { error: "name and city must be strings" });
        }
        const user = { id: nextId++, name: input.name, city: input.city };
        users.set(user.id, user);
        res.setHeader("Location", `/users/${user.id}`);
        return json(res, 201, user);
      } catch {
        return json(res, 400, { error: "Invalid JSON body" });
      }
    }
    res.setHeader("Allow", "GET, HEAD, POST, OPTIONS");
    return json(res, 405, { error: "Method not allowed" });
  }

  const match = /^\/users\/(\d+)$/.exec(pathname);
  if (match) {
    const id = Number(match[1]);
    const existing = users.get(id);
    if (method === "OPTIONS") {
      res.writeHead(204, { Allow: "GET, HEAD, PUT, PATCH, DELETE, OPTIONS" });
      return res.end();
    }
    if (method === "GET") {
      return existing ? json(res, 200, existing) : json(res, 404, { error: "Not found" });
    }
    if (method === "HEAD") {
      res.writeHead(existing ? 200 : 404);
      return res.end();
    }
    if (method === "DELETE") {
      if (!existing) return json(res, 404, { error: "Not found" });
      users.delete(id);
      res.writeHead(204);
      return res.end();
    }
    if (method === "PUT" || method === "PATCH") {
      if (!existing) return json(res, 404, { error: "Not found" });
      try {
        const input = await readBody(req);
        if (!input || typeof input !== "object" || Array.isArray(input) ||
            !Object.keys(input).every(k => ["name", "city"].includes(k))) {
          return json(res, 400, { error: "Invalid fields" });
        }
        if ((method === "PUT" &&
             (typeof input.name !== "string" || typeof input.city !== "string")) ||
            (method === "PATCH" &&
             Object.values(input).some(v => typeof v !== "string"))) {
          return json(res, 400, { error: "Invalid user data" });
        }
        const updated = method === "PUT"
          ? { id, name: input.name, city: input.city }
          : { ...existing, ...input };
        users.set(id, updated);
        return json(res, 200, updated);
      } catch {
        return json(res, 400, { error: "Invalid JSON body" });
      }
    }
    res.setHeader("Allow", "GET, HEAD, PUT, PATCH, DELETE, OPTIONS");
    return json(res, 405, { error: "Method not allowed" });
  }

  return json(res, 404, { error: "Route not found" });
});

server.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});
```

### How to test

Run the server first, then try the `fetch()` examples in Sections 1–7 using Node.js or your API client. User `1` already exists, so PUT, PATCH, and DELETE can target `/users/1`.

POST with `curl`:

```bash
curl -i -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Rahul","city":"Pune"}'
```

Expected: `201 Created`, JSON containing an `id`, and a `Location` header.

**How it works:** `req.method` identifies the method; `url.pathname` identifies the route; `for await (const chunk of req)` reads request-body chunks; `JSON.parse()` converts body text to an object; `res.end()` completes the response.

**For learning only:** A production server additionally needs body-size limits, stronger validation, persistence, authentication and robust error handling. `OPTIONS` here demonstrates `Allow`, not a complete CORS preflight configuration.

---

## 14. 405 Method Not Allowed

Suppose route exists:

```text
/users
```

but only supports GET.

If client sends:

```http
DELETE /users
```

a better response may be:

```text
405 Method Not Allowed
```

instead of 404.

This distinction matters.

---

## 15. Allow header

With 405 responses, servers may include:

```http
Allow: GET, POST
```

This tells the client which methods are supported.

---

## 16. REST mapping

Typical REST conventions:

```text
GET    /users
GET    /users/:id
POST   /users
PUT    /users/:id
PATCH  /users/:id
DELETE /users/:id
```

---

## 17. Common mistakes

### Mistake 1
Using GET to perform deletes or writes.

Bad API semantics.

### Mistake 2
Using POST for every endpoint.

Loses meaningful HTTP semantics.

### Mistake 3
Confusing PUT and PATCH.

### Mistake 4
Saying DELETE is not idempotent because second request returns 404.

Idempotency is about intended effect/state, not identical response code.

---

## 18. Interview questions

### GET vs POST?

GET retrieves resources and is safe/idempotent by semantic definition. POST commonly creates resources or performs non-idempotent actions.

### PUT vs PATCH?

PUT generally replaces the full target representation; PATCH applies partial changes.

### What is idempotency?

Repeating the same request has the same intended effect as executing it once.

### Is DELETE idempotent?

Yes, semantically.

---

## 19. Strong interview answer

> HTTP methods communicate operation semantics. GET retrieves resources and should be safe. POST is usually used for creation or non-idempotent actions. PUT represents full replacement semantics, PATCH partial modification, and DELETE removes a resource. Idempotency is important because retries happen in distributed systems; GET, PUT, and DELETE are semantically idempotent, while POST usually needs an explicit idempotency strategy for critical writes such as payments.

---

## Interview-Ready Summary

```text
GET
  -> read

POST
  -> create/action

PUT
  -> full replace

PATCH
  -> partial update

DELETE
  -> remove

HEAD
  -> GET metadata, no body

OPTIONS
  -> supported communication options

Interview:
safe vs idempotent
```

## Practice Task

Design method + endpoint combinations for:

- users
- orders
- profile update
- password reset
- payment creation
- payment retry

Explain idempotency for each.
