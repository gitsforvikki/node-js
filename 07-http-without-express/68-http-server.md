# Lesson 68 — Creating an HTTP Server

## Why this lesson matters

Creating a server with `node:http` shows exactly what frameworks are abstracting away.

You should be able to build a small API without Express.

---

## 1. Basic server

```js
import http from "node:http";

const server =
  http.createServer(
    (req, res) => {
      res.end("Hello Node.js");
    }
  );

server.listen(
  3000,
  () => {
    console.log(
      "Server running on port 3000"
    );
  }
);
```

---

## 2. What createServer does

Conceptually:

```text
createServer(handler)
       |
       v
HTTP server object
       |
       v
listen(port)
       |
       v
OS socket bound
       |
       v
incoming request
       |
       v
handler(req, res)
```

---

## 3. The handler is event-driven

The callback:

```js
(req, res) => {}
```

runs for every incoming request.

Internally, the server is event-driven.

You can also write:

```js
const server =
  http.createServer();

server.on(
  "request",
  (req, res) => {
    res.end("Hello");
  }
);
```

These are conceptually equivalent styles.

---

## 4. Simple routing

```js
const server =
  http.createServer(
    (req, res) => {
      if (
        req.method === "GET" &&
        req.url === "/"
      ) {
        res.statusCode = 200;
        return res.end("Home");
      }

      if (
        req.method === "GET" &&
        req.url === "/health"
      ) {
        res.statusCode = 200;

        res.setHeader(
          "Content-Type",
          "application/json"
        );

        return res.end(
          JSON.stringify({
            status: "ok",
          })
        );
      }

      res.statusCode = 404;
      res.end("Not Found");
    }
  );
```

This is manual routing.

---

## 5. Why manual routing becomes difficult

With many endpoints:

```text
GET /users
GET /users/:id
POST /users
PATCH /users/:id
DELETE /users/:id
GET /orders
POST /orders
...
```

manual if/else logic becomes messy.

That is one reason routing frameworks exist.

---

## 6. Parse URL properly

Do not rely only on raw `req.url`.

Example:

```js
const url =
  new URL(
    req.url,
    `http://${req.headers.host}`
  );

console.log(
  url.pathname
);
```

Now:

```text
/users?page=2
```

becomes:

```text
pathname: /users
page: 2
```

---

## 7. Setting JSON responses

Reusable helper:

```js
function sendJson(
  res,
  statusCode,
  data
) {
  res.writeHead(
    statusCode,
    {
      "Content-Type":
        "application/json; charset=utf-8",
    }
  );

  res.end(
    JSON.stringify(data)
  );
}
```

---

## 8. Writing status and headers

```js
res.writeHead(
  201,
  {
    "Content-Type":
      "application/json",
  }
);
```

Then:

```js
res.end(
  JSON.stringify({
    id: 1,
  })
);
```

---

## 9. `res.write()`, `res.end()`, `res.send()` and `res.json()` ⭐⭐⭐⭐⭐

These methods all relate to sending an HTTP response, but **not all of them belong to Node.js itself**.

> **Key difference:** `res.write()` and `res.end()` come from Node's built-in `node:http` response object. `res.send()` and `res.json()` are conveniences added by **Express.js**.

### 9.1 `res.end()` — Send final data and finish the response

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  res.end("Hello Node.js");
});

server.listen(3000);
```

When the browser requests `http://localhost:3000`, `res.end("Hello Node.js")` sends the text and **marks the response as complete**.

```js
res.end("Hello"); // Sends "Hello" and finishes
res.end("World"); // ❌ Error: response has already ended
```

You can also use `res.end()` with no argument after previously writing body data.

### 9.2 `res.write()` — Write chunks without finishing

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  res.write("Hello ");
  res.write("Vikash ");
  res.write("Welcome to Node.js");

  res.end();
});

server.listen(3000);
```

The response body contains:

```text
Hello Vikash Welcome to Node.js
```

Flow:

```text
res.write("Hello ")       → write first body chunk
res.write("Vikash ")      → write another body chunk
res.write("Welcome ...")  → write another body chunk
res.end()                 → finish HTTP response
```

`res.write()` **does not finish** the response. Use it for incremental output or streaming, and eventually call `res.end()`. Writes can be buffered by Node, proxies, or clients; calling `write()` does not guarantee each chunk appears in the browser immediately.

### 9.3 `res.send()` — An Express method, not a native HTTP method

Express example:

```js
import express from "express";

const app = express();

app.get("/", (req, res) => {
  res.send("Hello Express");
});

app.listen(3000);
```

`res.send()` sends the response, handles relevant response headers, and **finishes** it automatically. You do not need to call `res.end()` after it.

In raw `node:http`, however:

```js
import http from "node:http";

http.createServer((req, res) => {
  res.send("Hello"); // ❌ TypeError: res.send is not a function
}).listen(3000);
```

Why? Node's `http.ServerResponse` has no `send()` method. Express extends the response interface with helpers.

### 9.4 `res.json()` — Convenient JSON responses in Express

```js
app.get("/user", (req, res) => {
  res.status(200).json({
    name: "Vikash",
    role: "Developer",
  });
});
```

Express serializes the object to JSON, sets an appropriate JSON content type, and ends the response. `res.send({ name: "Vikash" })` also sends JSON in Express, but `res.json()` makes the API's intent clearer.

### 9.5 Raw Node.js vs Express — Same JSON response

**Raw Node.js HTTP server:**

```js
import http from "node:http";

http.createServer((req, res) => {
  res.setHeader("Content-Type", "application/json");

  res.end(JSON.stringify({
    name: "Vikash",
  }));
}).listen(3000);
```

**Express server:**

```js
import express from "express";

const app = express();

app.get("/", (req, res) => {
  res.json({ name: "Vikash" });
});

app.listen(3000);
```

Raw Node.js requires you to serialize the object and set the header yourself; Express provides a higher-level helper.

### 9.6 What about `res.writeHead()`?

In raw Node.js, `writeHead()` sets the response status code and headers:

```js
res.writeHead(200, {
  "Content-Type": "text/html",
});

res.end("<h1>Hello Vikash</h1>");
```

`writeHead()` **does not end** the response. You must still send/end it.

### 9.7 Comparison and when to use each

| Method | Built-in `node:http` | Express | Ends response? |
| --- | --- | --- | --- |
| `res.write()` | ✅ | ✅ | No |
| `res.end()` | ✅ | ✅ | Yes |
| `res.send()` | ❌ | ✅ | Yes |
| `res.json()` | ❌ | ✅ | Yes |
| `res.writeHead()` | ✅ | ✅ | No |

Express builds on Node's response object, so the low-level Node methods remain available.

**Practical choice:**

- **Simple text response with raw Node:** `res.end("Hello")`.
- **Multiple chunks / streaming with raw Node:** `res.write(chunk)` followed by `res.end()`.
- **Text or HTML with Express:** `res.send(content)`.
- **JSON APIs with Express:** `res.json(data)`.
- **Empty response with Express:** `res.status(204).end()`.

**Interview mental model:**

```text
res.write() → Add body data; keep the response open.
res.end()   → Finish the response (optionally send final data).
res.send()  → Express sends data and finishes automatically.
res.json()  → Express serializes JSON and finishes automatically.
```

---

## 10. Hanging response bug

Bad:

```js
if (req.url === "/users") {
  res.write("users");
}
```

If you never call `res.end()`, the client may wait indefinitely.

---

## 11. Server errors

```js
server.on(
  "error",
  (error) => {
    console.error(
      "Server error",
      error
    );
  }
);
```

Example startup error:

```text
EADDRINUSE
```

means the port is already in use.

---

## 12. Graceful shutdown

```js
process.on(
  "SIGTERM",
  () => {
    server.close(
      () => {
        console.log(
          "Server closed"
        );
      }
    );
  }
);
```

Production flow:

```text
SIGTERM
   |
   v
stop accepting new requests
   |
   v
finish active requests
   |
   v
close DB/Redis
   |
   v
exit
```

---

## 13. Server timeouts

Production servers should think about:

- request timeout
- headers timeout
- keep-alive timeout

Why?

Slow or malicious clients can otherwise hold connections for too long.

Exact configuration depends on runtime and infrastructure.

---

## 14. Listen on host

```js
server.listen(
  3000,
  "0.0.0.0"
);
```

In containers, binding to `0.0.0.0` is often required so traffic can reach the process externally.

Binding only to localhost can make a containerized app unreachable from outside the container.

---

## 15. Environment-based port

```js
const PORT =
  Number(
    process.env.PORT ?? 3000
  );
```

This is standard production practice.

---

## 16. Common mistakes

### Mistake 1
Forgetting `res.end()`.

### Mistake 2
Using hardcoded port only.

### Mistake 3
Ignoring server error events.

### Mistake 4
No graceful shutdown.

### Mistake 5
Manual routing without URL parsing.

---

## 17. Interview questions

### What does http.createServer return?

An HTTP server object that listens for incoming requests.

### What are req and res?

The incoming request and outgoing response objects.

### Why is graceful shutdown important?

To stop taking new traffic while allowing in-flight work and resources to close cleanly.

### Why might 0.0.0.0 matter in Docker?

It binds on all network interfaces instead of only localhost.

---

## 18. Strong interview answer

> A raw Node.js HTTP server is created with http.createServer. The request handler receives IncomingMessage and ServerResponse objects. The server binds to a port using listen(), and every incoming HTTP request triggers the request handler. In production, I also handle server errors, configurable ports, timeouts, and graceful shutdown rather than only calling listen().

---

## Interview-Ready Summary

```text
http.createServer()
       |
       v
request handler
       |
       +--> inspect method/url
       +--> set status
       +--> set headers
       +--> write body
       +--> end response

Production:
errors
timeouts
graceful shutdown
env port
```

## Practice Task

Build a raw Node.js server with:

- GET /
- GET /health
- GET /users

Return JSON and a proper 404 for unknown routes.
