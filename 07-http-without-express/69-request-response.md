# Lesson 69 — Request and Response Objects

## Why this lesson matters

Most web frameworks expose objects called:

```text
req
res
```

Those concepts come from Node's raw HTTP layer.

Understanding them helps you debug framework behavior and answer low-level interview questions.

---

## 1. Request object

In a Node HTTP handler:

```js
(req, res) => {}
```

`req` is an `IncomingMessage`.

It contains:

- method
- URL
- headers
- HTTP version
- socket information
- request body stream

---

## 2. Request method

```js
console.log(
  req.method
);
```

Examples:

```text
GET
POST
PUT
PATCH
DELETE
OPTIONS
HEAD
```

---

## 3. Request URL

```js
console.log(
  req.url
);
```

Example:

```text
/users?page=2
```

This includes path/query, not the complete absolute URL in the typical server request form.

---

## 4. Request headers

```js
console.log(
  req.headers
);
```

Examples:

```text
host
content-type
authorization
user-agent
accept
cookie
```

Header names are generally exposed normalized to lowercase in Node's request headers object.

---

## 5. Request body is a Readable stream

Example:

```js
const chunks = [];

req.on(
  "data",
  (chunk) => {
    chunks.push(chunk);
  }
);

req.on(
  "end",
  () => {
    const body =
      Buffer.concat(
        chunks
      );

    console.log(body);
  }
);
```

This is the raw request body.

---

## 6. Parse JSON safely

```js
let body = "";

req.setEncoding("utf8");

req.on(
  "data",
  (chunk) => {
    body += chunk;
  }
);

req.on(
  "end",
  () => {
    try {
      const data =
        JSON.parse(body);

      console.log(data);
    } catch {
      res.statusCode = 400;
      res.end(
        "Invalid JSON"
      );
    }
  }
);
```

---

## 7. Body-size limits

Never accept unlimited request bodies.

Danger:

```text
attacker sends
5 GB JSON body
   |
   v
server keeps buffering
   |
   v
memory exhaustion
```

You should enforce a limit.

Conceptual:

```js
let size = 0;

req.on(
  "data",
  (chunk) => {
    size += chunk.length;

    if (
      size > MAX_BODY_SIZE
    ) {
      req.destroy();
    }
  }
);
```

Frameworks make this easier.

---

## 8. Socket information

```js
console.log(
  req.socket.remoteAddress
);
```

This can give connection-level information.

But behind proxies/load balancers, client IP requires trusted proxy configuration.

Do not blindly trust forwarded headers.

---

# Response object

`res` is a `ServerResponse`.

It is a Writable stream.

---

## 9. Set status code

```js
res.statusCode = 201;
```

or:

```js
res.writeHead(201);
```

---

## 10. Set headers

```js
res.setHeader(
  "Content-Type",
  "application/json"
);
```

Read:

```js
res.getHeader(
  "Content-Type"
);
```

---

## 11. Send body

```js
res.end(
  JSON.stringify({
    success: true,
  })
);
```

---

## 12. Streaming response

Because `res` is Writable:

```js
res.write("chunk 1");
res.write("chunk 2");
res.end();
```

This is useful for:
- large responses
- streaming data
- progressive output

---

## 13. Headers must be sent before body

Once headers are sent, changing them may fail.

Bad:

```js
res.write("hello");

res.setHeader(
  "Content-Type",
  "application/json"
);
```

You generally configure status/headers first.

---

## 14. headersSent

```js
console.log(
  res.headersSent
);
```

Useful when debugging double-response problems.

---

## 15. Double response bug

Bad:

```js
if (!user) {
  res.statusCode = 404;
  res.end("Not found");
}

res.end("Success");
```

If user is missing, code continues and tries to send again.

Correct:

```js
if (!user) {
  res.statusCode = 404;

  return res.end(
    "Not found"
  );
}
```

---

## 16. Request aborted / client disconnect

Clients can disconnect before processing finishes.

For long operations, production systems should think about:
- cancellation
- abort signals
- avoiding wasted work

---

## 17. Common mistakes

### Mistake 1
Assuming req.body exists in raw Node.

It does not.

### Mistake 2
Ignoring request-size limits.

### Mistake 3
Trying to modify headers after body has started.

### Mistake 4
Sending multiple responses.

### Mistake 5
Trusting forwarded IP headers without proxy configuration.

---

## 18. Interview questions

### What type of object is req?

An IncomingMessage, which is also a Readable stream.

### What type is res?

A ServerResponse, which behaves as a Writable stream.

### Why is the body streamed?

Because network data arrives incrementally rather than necessarily as one complete in-memory object.

### What causes "headers already sent"?

Trying to modify/send another response after headers/body have already started.

---

## 19. Strong interview answer

> In raw Node.js, req is an IncomingMessage and acts as a Readable stream, so request bodies arrive as chunks. res is a ServerResponse and acts as a Writable stream. The request exposes method, URL, headers, socket, and body data, while the response controls status, headers, and body output. Frameworks such as Express add conveniences like req.body and res.json on top of these primitives.

---

## Interview-Ready Summary

```text
req
  -> IncomingMessage
  -> Readable stream
  -> method
  -> url
  -> headers
  -> body chunks

res
  -> ServerResponse
  -> Writable stream
  -> status
  -> headers
  -> body
```

## Practice Task

Create a POST /echo route that:

1. accepts JSON
2. limits body size
3. returns 400 for invalid JSON
4. echoes valid input
5. prevents double responses
