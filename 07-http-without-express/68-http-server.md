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

## 9. res.write() vs res.end()

```js
res.write("part 1");
res.write("part 2");
res.end("done");
```

`write()` sends body chunks.

`end()` finishes the response.

You must eventually end the response.

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
