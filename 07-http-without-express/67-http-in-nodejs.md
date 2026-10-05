# Lesson 67 — How HTTP Works in Node.js

## Why this lesson matters

Before using Express, Fastify, NestJS, or any higher-level framework, you should understand what Node.js is actually doing underneath.

If you understand raw HTTP in Node.js, you can explain:

- what happens when a browser calls your API
- what request and response really are
- how sockets, headers, methods, and bodies fit together
- why HTTP is stateless
- how Node.js maps network traffic into JavaScript objects
- what Express is abstracting away

This is highly valuable in interviews.

---

## 1. What is HTTP?

HTTP stands for:

```text
Hypertext Transfer Protocol
```

It is an application-layer protocol used for communication between clients and servers.

Typical flow:

```text
Client
   |
   | HTTP Request
   v
Server
   |
   | HTTP Response
   v
Client
```

---

## 2. HTTP runs over a transport connection

For classic HTTP/1.1 and HTTP/2, communication commonly runs over TCP.

Conceptually:

```text
HTTP
   |
   v
TCP
   |
   v
IP
   |
   v
Network
```

HTTPS adds TLS:

```text
HTTP
   |
   v
TLS
   |
   v
TCP
```

Node.js exposes different core modules depending on protocol needs, such as:
- `node:http`
- `node:https`
- `node:http2`

---

## 3. What an HTTP request contains

A request generally includes:

```text
METHOD path HTTP/version
Headers

Optional Body
```

Example:

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer token

{"name":"Vikash"}
```

---

## 4. What an HTTP response contains

A response generally includes:

```text
HTTP/version status-code status-text
Headers

Optional Body
```

Example:

```http
HTTP/1.1 201 Created
Content-Type: application/json

{"id":1,"name":"Vikash"}
```

---

## 5. HTTP request-response lifecycle in Node.js

Mental model:

```text
Client
   |
   v
TCP connection
   |
   v
Node HTTP parser/runtime
   |
   v
IncomingMessage object
   |
   v
your request handler
   |
   v
ServerResponse object
   |
   v
bytes sent back to client
```

---

## 6. IncomingMessage and ServerResponse

When you create a server:

```js
import http from "node:http";

const server =
  http.createServer(
    (req, res) => {
      // req = incoming request
      // res = outgoing response
    }
  );
```

`req` is an instance based on Node's `IncomingMessage`.

`res` is a `ServerResponse`.

These are not Express objects.

Express wraps and enhances them.

---

## 7. HTTP is stateless

HTTP does not automatically remember previous requests.

Request 1:

```text
GET /profile
```

Request 2:

```text
GET /orders
```

By default, the server does not inherently know they belong to the same user.

State is added using things such as:

- cookies
- sessions
- authorization tokens
- API keys

---

## 8. Persistent connections

HTTP/1.1 commonly supports connection reuse.

Instead of:

```text
request
new TCP connection
response
close
```

every time, a connection may be reused:

```text
TCP connection
   |
   +--> request 1 / response 1
   +--> request 2 / response 2
   +--> request 3 / response 3
```

This reduces connection setup overhead.

---

## 9. HTTP body arrives as a stream

This is very important in Node.js.

The request body is not magically one complete object.

`req` is a Readable stream.

Example:

```js
let body = "";

req.on("data", (chunk) => {
  body += chunk;
});

req.on("end", () => {
  console.log(body);
});
```

For binary payloads, chunks may be Buffers.

---

## 10. Why frameworks feel easier

Raw Node.js requires you to handle:

- routing
- body parsing
- status codes
- headers
- errors
- middleware behavior
- content negotiation

Express adds abstractions for these.

Mental model:

```text
Express
   |
   v
node:http
   |
   v
TCP/network stack
```

---

## 11. HTTP message boundaries

You should not think:

> one TCP packet equals one HTTP request

That is wrong.

TCP is a byte stream.

An HTTP message may be split across multiple packets, and Node handles protocol parsing before your application sees the request object.

---

## 12. Content-Length and transfer framing

HTTP needs to know where a body ends.

Common mechanisms include:

- `Content-Length`
- chunked transfer encoding in HTTP/1.1
- protocol framing in newer versions

Application frameworks usually hide these low-level details.

---

## 13. HTTPS

HTTPS means HTTP over TLS.

It provides:

- encryption
- integrity
- server authentication

Do not say:

> HTTPS is a different application protocol from HTTP

A better explanation is:

> HTTPS is HTTP secured with TLS.

---

## 14. Real API lifecycle

```text
Browser
   |
   v
DNS resolves domain
   |
   v
TCP/TLS connection
   |
   v
HTTP request
   |
   v
Node server
   |
   v
route logic
   |
   v
database/API
   |
   v
HTTP response
   |
   v
browser receives data
```

---

## 15. Common mistakes

### Mistake 1
Thinking Express itself implements HTTP from scratch.

It sits on top of Node's HTTP primitives.

### Mistake 2
Thinking request body is already parsed JSON.

Raw Node.js gives you bytes/chunks.

### Mistake 3
Thinking HTTP remembers users automatically.

HTTP is stateless.

### Mistake 4
Thinking one request equals one TCP packet.

TCP is a stream.

---

## 16. Interview questions

### What happens when a client calls a Node.js HTTP server?

The request arrives through the network connection, Node's HTTP layer parses it, creates an IncomingMessage object, invokes your handler, and your ServerResponse writes the response back over the connection.

### Is HTTP stateful?

No. State is usually added through cookies, sessions, or tokens.

### Is the request body immediately available as JSON?

No. In raw Node.js, the request body arrives as a stream of chunks.

### What does Express add?

Routing, middleware, request parsing, response helpers, error handling, and developer-friendly abstractions over node:http.

---

## 17. Strong interview answer

> HTTP is a request-response application protocol. In Node.js, the node:http module accepts network connections, parses incoming HTTP messages, and exposes them as IncomingMessage objects. The request body is streamed, while ServerResponse is used to send status, headers, and body data back to the client. HTTP itself is stateless, so authentication state is usually implemented with cookies, sessions, or tokens.

---

## Interview-Ready Summary

```text
HTTP Request
   |
   +--> method
   +--> path
   +--> headers
   +--> optional body

Node.js
   |
   +--> IncomingMessage
   +--> ServerResponse
   +--> streaming body
   +--> raw HTTP primitives

HTTP
   -> stateless
   -> request/response
   -> usually over TCP/TLS
```

## Practice Task

Open browser DevTools or curl and inspect a real API request.

Identify:
- method
- URL
- request headers
- request body
- response status
- response headers
- response body
