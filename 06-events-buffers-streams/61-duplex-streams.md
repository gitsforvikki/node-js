# Lesson 61 — Duplex Streams

## Why this lesson matters

A Duplex stream can both **read data and write data**.

This is important because many real Node.js objects are naturally two-way communication channels.

Examples include:

- TCP sockets
- TLS sockets
- some compression or protocol layers
- custom bidirectional stream abstractions

---

## 1. What is a Duplex stream?

A Duplex stream combines:

- a Readable side
- a Writable side

Mental model:

```text
        Duplex Stream
      <-------------->
Readable side    Writable side
```

That means data can flow in both directions.

---

## 2. Real-world example — TCP socket

A TCP socket can:

- receive bytes
- send bytes

```text
Client
  |
  | send data
  v
Socket
  |
  | receive data
  v
Server

Server
  |
  | send response
  v
Socket
  |
  | receive response
  v
Client
```

This is a natural Duplex model.

---

## 3. Basic custom Duplex example

```js
import {
  Duplex,
} from "node:stream";

const duplex =
  new Duplex({
    read() {
      this.push("Hello");
      this.push(null);
    },

    write(
      chunk,
      encoding,
      callback
    ) {
      console.log(
        "Received:",
        chunk.toString()
      );

      callback();
    },
  });
```

You can write into it:

```js
duplex.write("Input data");
```

And read from it:

```js
duplex.on("data", (chunk) => {
  console.log(
    "Output:",
    chunk.toString()
  );
});
```

---

## 4. Read and write sides are separate

This is very important.

In a Duplex stream:

```text
Readable side
   !=
Writable side
```

Writing data does not automatically mean that same data appears on the readable side.

If you want input transformed into output, that is usually a **Transform stream**, covered next.

---

## 5. Duplex lifecycle

Readable side may end:

```js
this.push(null);
```

Writable side may finish when:

```js
duplex.end();
```

These are separate lifecycle concepts.

---

## 6. allowHalfOpen

Duplex streams can have one side close while the other remains open.

This is called a half-open state.

Conceptually:

```text
Readable side: closed
Writable side: still open
```

Node.js Duplex streams support configuration around this behavior.

This is especially relevant in socket programming.

---

## 7. TCP socket example

```js
import net from "node:net";

const server =
  net.createServer(
    (socket) => {
      socket.on(
        "data",
        (chunk) => {
          console.log(
            chunk.toString()
          );

          socket.write(
            "Message received"
          );
        }
      );
    }
  );
```

`socket` is Duplex:

- `data` comes from readable side
- `write()` uses writable side

---

## 7A. Hands-on: Server and client in two terminals

A TCP socket is a **Duplex stream**: each connected socket can both receive data (`data` event) and send data (`write()`).

**Real-life analogy:** A phone call lets you listen and speak over the same connection.

Create **two separate files**.

### server.js

```js
import net from "node:net";

const streamServer = net.createServer((socket) => {
  console.log("Client connected");

  // READ: receive data from client
  socket.on("data", (data) => {
    console.log("Client:", data.toString());

    // WRITE: send data back to client
    socket.write("Hello from server...");
  });

  socket.on("end", () => {
    console.log("Client finished sending data");
  });

  socket.on("error", (error) => {
    console.error("Socket error:", error.message);
  });
});

streamServer.listen(3000, () => {
  console.log("Server is running at 3000");
});
```

### client.js

```js
import net from "node:net";

const client = net.createConnection({ port: 3000 }, () => {
  // WRITE: send data to the server
  client.write("Hello server,what are you doing...");
});

// READ: receive server response
client.on("data", (data) => {
  console.log("From Server:", data.toString());
  client.end(); // no more data to send
});

client.on("error", (error) => {
  console.error("Connection error:", error.message);
});
```

> These examples use ES modules: use `"type": "module"` in `package.json`, or name the files `.mjs`.

### How to run: open **two terminals**

**Terminal 1 — server** (keep this running):

```bash
node server.js
```

Initially you only see:

```text
Server is running at 3000
```

This is normal: the server is listening but no client has connected yet.

**Terminal 2 — client** (run separately):

```bash
node client.js
```

**Server terminal after the client runs:**

```text
Server is running at 3000
Client connected
Client: Hello server,what are you doing...
Client finished sending data
```

**Client terminal:**

```text
From Server: Hello from server...
```

The client exits after calling `end()` and completing the connection shutdown. The server keeps listening for new clients.

### Step-by-step execution

1. `server.listen(3000)` starts waiting for connections.
2. `net.createConnection({ port: 3000 })` connects the client.
3. The client's `write()` sends bytes **to** the server.
4. The server's `socket.on("data")` receives and logs those bytes.
5. The server's `socket.write()` sends a reply **to** the client.
6. The client's `on("data")` receives the reply and calls `end()`.
7. The server's `end` event means the client finished its writable side; it does not mean the server stopped listening.

```text
CLIENT SOCKET                       SERVER SOCKET
  write("Hello server") ----------> on("data") [READ]
  on("data") [READ] <------------- write("Hello from server")
  end() --------------------------> on("end")
```

**Key distinction:** `net.createServer()` creates the listening server, while `net.createConnection()` creates a separate client connection. Running only `server.js` does not send a client message. Both ends expose Duplex sockets.

**Also remember:** TCP transfers a stream of bytes, not guaranteed one-message-per-`data`-event packets. For real applications, define message boundaries (for example, newline-delimited messages).

---

## 8. Backpressure on Duplex streams

Because Duplex streams have both sides, backpressure considerations can apply independently.

Writable side:

```js
const ok =
  socket.write(data);

if (!ok) {
  // wait for drain
}
```

Readable side:
- can be paused
- resumed
- piped downstream

---

## 9. Duplex vs Transform

```text
Duplex
  readable and writable are independent

Transform
  writable input produces readable output
```

Example:

TCP socket:

```text
write outbound bytes
read inbound bytes
```

Transform stream:

```text
input text
   |
   v
uppercase transform
   |
   v
output text
```

---

## 10. PassThrough stream

Node.js provides a special Transform stream called `PassThrough`.

It simply forwards input to output.

Conceptually:

```text
input
  |
  v
PassThrough
  |
  v
same output
```

Useful for:
- inspection
- metrics
- debugging
- piping duplication patterns

---

## 11. Common mistakes

### Mistake 1
Assuming Duplex automatically transforms written data into readable data.

Wrong.

### Mistake 2
Confusing Duplex with Transform.

Transform is a specialized Duplex stream.

### Mistake 3
Ignoring half-open behavior in sockets.

### Mistake 4
Ignoring backpressure on the writable side.

---

## 12. Interview questions

### What is a Duplex stream?

A stream that has both readable and writable interfaces.

### Are the readable and writable sides connected automatically?

No.

### Give a real Node.js example.

A TCP socket.

### How is Transform different?

A Transform stream is a Duplex stream where written input is processed to produce readable output.

---

## 13. Strong interview answer

> A Duplex stream combines a Readable and Writable stream in one object. The two sides are independent, which makes Duplex streams suitable for bidirectional communication such as TCP sockets. A Transform stream is a specialized Duplex stream where writable input is intentionally converted into readable output.

---

## Interview-Ready Summary

```text
Duplex
   |
   +--> readable side
   +--> writable side
   +--> independent directions

Example:
TCP socket

Transform
   -> specialized Duplex
   -> input becomes output
```

## Practice Task

Build a small TCP echo server using `node:net`.

Explain:
- which side is readable
- which side is writable
- how backpressure could occur
