# Lesson 28 — http

## Deep Dive — Kya req.on("data") aur req.on("end") Events Hain?

**Haan!** Ye Node.js ke real-world **EventEmitter + Readable Stream** ka example hai. Is section mein har line, output aur event flow ko samjhenge.

### 1. Sabse pehle code samjho

```js
let body = "";

req.on("data", (chunk) => {
  body += chunk;
});

req.on("end", () => {
  console.log(body);
});
```

- **req**: HTTP request object, jo **Readable Stream** bhi hai.
- **on()**: Event listener register karta hai.
- **data**: Jab request body ka chunk read hone ke liye available hota hai, tab event fire hota hai.
- **end**: Jab stream ka saara data read ho chuka ho, tab event fire hota hai.

### 2. Real-life POST Request Example

Maan lo React frontend se POST request bhejte ho:

```js
fetch("http://localhost:3000/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    name: "Vikash",
    age: 25,
  }),
});
```

Frontend se logically ye JSON send hota hai:

```json
{
  "name": "Vikash",
  "age": 25
}
```

Node.js request body ko stream ki tarah read karta hai. Data **ek ya multiple chunks** mein aa sakta hai.

**Illustrative graphics — 3 chunks (actual chunks alag ho sakte hain):**

```mermaid
flowchart TD
    A["HTTP POST Request"] --> B["Readable request stream: req"]
    B --> C["Chunk 1: {\"name\":"]
    B --> D["Chunk 2: \"Vikash\","]
    B --> E["Chunk 3: \"age\":25}"]
    C --> F["data event → body += chunk"]
    D --> F
    E --> F
    F --> G["end event → console.log(body)"]
```

GitHub Mermaid diagram ko graphics ki tarah render karta hai. Agar diagram render na ho, ye plain-text version dekho:

```text
HTTP POST request
        |
        v
req (Readable Stream)
        |
        +-- chunk 1 --> data event --> body += chunk
        +-- chunk 2 --> data event --> body += chunk
        +-- chunk 3 --> data event --> body += chunk
        |
        +-- stream finished --> end event --> console.log(body)
```

**Important:** Network/stream buffering ki wajah se exactly 3 chunks milna guaranteed nahi hai. Ek chunk bhi aa sakta hai, ya bahut saare.

### 3. Step-by-step Execution with Output

```js
let body = "";

req.on("data", (chunk) => {
  body += chunk;
});
```

Har `data` event ke saath next chunk listener ko milta hai. Maan lo chunks aise mile:

```text
First data event:
chunk = '{"name":'
body  = '{"name":'

Second data event:
chunk = '"Vikash",'
body  = '{"name":"Vikash",'

Third data event:
chunk = '"age":25}'
body  = '{"name":"Vikash","age":25}'
```

Jab stream successfully finish hoti hai:

```js
req.on("end", () => {
  console.log(body);
});
```

**Output:**

```text
{"name":"Vikash","age":25}
```

### 4. data aur end Event Emit Kaun Karta Hai?

Pichhle EventEmitter lesson mein humne manually event trigger kiya:

```js
emitter.emit("greet");
```

Lekin HTTP example mein hum sirf listeners likhte hain:

```js
req.on("data", callback);
req.on("end", callback);
```

**Node.js ka internal stream system ye events emit karta hai.** `req` ek `http.IncomingMessage` object hai jo `Readable` stream extend karta hai. Readable streams EventEmitter se functionality inherit karti hain.

| Custom EventEmitter | HTTP Request Stream |
| --- | --- |
| Hum event name choose karte hain | Node.js built-in stream events provide karta hai |
| Hum `emit()` call karte hain | Stream internally events emit karti hai |
| Hum `on()` se listen karte hain | Hum `req.on()` se listen karte hain |

### 5. Complete Working HTTP Server Example

**server.js**

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  if (req.method === "POST" && req.url === "/users") {
    let body = "";

    req.on("data", (chunk) => {
      console.log("Data event fired");
      body += chunk;
    });

    req.on("end", () => {
      console.log("End event fired");

      let user;
      try {
        user = JSON.parse(body);
      } catch {
        res.writeHead(400, { "Content-Type": "text/plain" });
        return res.end("Invalid JSON");
      }

      console.log("User:", user);

      res.writeHead(201, {
        "Content-Type": "application/json",
      });

      res.end(JSON.stringify({
        success: true,
        user,
      }));
    });

    req.on("error", (error) => {
      console.error("Request stream error:", error);
      if (!res.headersSent) {
        res.writeHead(400);
      }
      if (!res.writableEnded) res.end("Request failed");
    });

    return;
  }

  res.writeHead(404);
  res.end("Not found");
});

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

Run:

```bash
node server.js
```

Then frontend `fetch()` example run karo, ya terminal mein:

```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Vikash","age":25}'
```

**Possible terminal output (agar body ek chunk mein aayi):**

```text
Server running on port 3000
Data event fired
End event fired
User: { name: 'Vikash', age: 25 }
```

**HTTP response body:**

```json
{
  "success": true,
  "user": {
    "name": "Vikash",
    "age": 25
  }
}
```

**Production note:** Ye learning example hai. Production mein request body size limit, invalid JSON, aborted requests aur response lifecycle ko robustly handle karna zaroori hai. Unlimited body accumulation memory exhaust kar sakta hai.

### 6. data vs end — Important Difference

| `data` event | `end` event |
| --- | --- |
| Chunk available hone par fire hota hai | Readable stream successfully finish hone par fire hota hai |
| Zero, one ya multiple times ho sakta hai | Normal successful completion par once hota hai |
| Callback ko chunk milta hai | Batata hai ki aur data nahi aayega |

Agar stream prematurely destroy/fail ho, `end` event aana guaranteed nahi hai.

### 7. Interview Question

**Q: Why do we use `req.on("data")` and `req.on("end")` in Node.js?**

Because an HTTP request is a readable stream. The `data` event lets us receive chunks of incoming request-body data, and the `end` event tells us the stream has finished providing data.

**Ek line mein yaad rakho:**

- `req.on("data")` = **Jab bhi data aaye, mujhe do.**
- `req.on("end")` = **Jab poora data aa jaye, mujhe batao.**

### 8. Express Mein Ye Manually Kyun Nahi Karte?

Express mein `express.json()` middleware request body ko read aur JSON parse karne ka kaam karta hai:

```js
import express from "express";

const app = express();

app.use(express.json());

app.post("/users", (req, res) => {
  console.log(req.body);
  res.status(201).json({ success: true, user: req.body });
});

app.listen(3000);
```

Isliye Express controller mein normally manually `req.on("data")` / `req.on("end")` likhne ki zaroorat nahi hoti. Express ke neeche wahi Node.js HTTP request stream hoti hai.

---

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
