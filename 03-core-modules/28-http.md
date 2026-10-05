# Lesson 28 — http

## Why Learn node:http?

Frameworks such as Express hide many details.

Learning `node:http` shows what actually happens underneath.

Import:

```js
import http from "node:http";
```

## Create a Server

```js
const server = http.createServer((req, res) => {
  res.end("Hello Node.js");
});

server.listen(3000);
```

Flow:

```text
Client
  |
  v
TCP/HTTP request
  |
  v
Node HTTP server
  |
  v
request handler
  |
  v
response
```

## Request Object

`req` contains information such as:

```js
req.method
req.url
req.headers
```

Example:

```js
console.log(req.method);
console.log(req.url);
console.log(req.headers);
```

## Response Object

Set status:

```js
res.statusCode = 200;
```

Set header:

```js
res.setHeader(
  "Content-Type",
  "application/json"
);
```

Send:

```js
res.end(JSON.stringify({
  success: true,
}));
```

## Basic Routing

```js
const server = http.createServer((req, res) => {
  if (
    req.method === "GET" &&
    req.url === "/health"
  ) {
    res.writeHead(200, {
      "Content-Type": "application/json",
    });

    return res.end(
      JSON.stringify({ status: "ok" })
    );
  }

  res.statusCode = 404;
  res.end("Not Found");
});
```

## Request Body Is a Stream

This is important.

`req` is a readable stream.

```js
let body = "";

req.on("data", (chunk) => {
  body += chunk;
});

req.on("end", () => {
  console.log(body);
});
```

For production code, body-size limits matter. Never accept unlimited request bodies.

## Parse JSON

```js
let body = "";

req.on("data", (chunk) => {
  body += chunk;
});

req.on("end", () => {
  try {
    const data = JSON.parse(body);
    console.log(data);
  } catch {
    res.statusCode = 400;
    return res.end("Invalid JSON");
  }
});
```

## Status Codes

Common examples:

```text
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Content
500 Internal Server Error
```

## Headers

Examples:

```js
res.setHeader(
  "Cache-Control",
  "no-store"
);

res.setHeader(
  "Content-Type",
  "application/json"
);
```

Headers communicate metadata about the message.

## Keep-Alive

HTTP connections may be reused for multiple requests, reducing connection setup overhead.

Modern Node.js HTTP behavior and defaults can evolve by version, so production tuning should match the runtime version.

## Timeouts Matter

Without correct timeout strategies, slow or malicious clients can consume resources.

Production servers should consider:
- request timeout
- headers timeout
- keep-alive timeout
- reverse proxy timeout configuration

## Graceful Shutdown

When shutting down:

```js
server.close(() => {
  console.log("Server closed");
});
```

A production shutdown flow often:
1. stops accepting new traffic
2. waits for active requests
3. closes DB/Redis connections
4. exits cleanly

## Express Relationship

Express sits on top of Node HTTP concepts.

```text
Express
   |
   v
node:http
   |
   v
TCP/networking
```

If you understand `node:http`, Express becomes much easier to understand.

## Interview Questions

### What are req and res in a Node HTTP server?

They are request and response objects. The request is also a readable stream, and the response is a writable stream.

### Why does Express make development easier?

It provides routing, middleware, body parsing, error handling and abstractions over lower-level HTTP APIs.

## Summary

```text
node:http
   |
   +--> server
   +--> request
   +--> response
   +--> headers
   +--> status
   +--> streams
```

## Practice Task

Build a mini REST API without Express:

```text
GET  /users
POST /users
GET  /health
```

Requirements:
- JSON responses
- status codes
- request body parsing
- 404 response
