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
