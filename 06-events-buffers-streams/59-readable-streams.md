# Lesson 59 — Readable Streams

## What is a Readable stream?

A Readable stream represents a source of data that can be consumed incrementally.

Examples:

- `fs.createReadStream()`
- HTTP request objects
- `process.stdin`
- custom Readable streams

---

## 1. Basic file stream

```js
import fs from "node:fs";

const stream =
  fs.createReadStream(
    "data.txt"
  );
```

---

## 2. data event

```js
stream.on(
  "data",
  (chunk) => {
    console.log(chunk);
  }
);
```

Once a `data` listener is attached, the stream commonly enters flowing mode.

---

## 3. Buffer chunks

By default, chunks are often Buffers.

```js
stream.on(
  "data",
  (chunk) => {
    console.log(
      Buffer.isBuffer(chunk)
    );
  }
);
```

---

## 4. Set encoding

Instead of receiving Buffers:

```js
stream.setEncoding("utf8");
```

Then chunks are strings.

---

## 5. end event

```js
stream.on(
  "end",
  () => {
    console.log(
      "No more data"
    );
  }
);
```

This means the readable side has finished producing data.

---

## 6. error event

```js
stream.on(
  "error",
  (error) => {
    console.error(error);
  }
);
```

Never ignore stream errors in production.

---

## 7. close event

```js
stream.on(
  "close",
  () => {
    console.log("closed");
  }
);
```

`end` and `close` are not identical concepts.

- `end`: no more readable data
- `close`: underlying resource closed

---

## 8. Flowing mode

When flowing, data is pushed automatically through `data` events.

```text
Readable
   |
   v
data
data
data
data
```

This is convenient but can be risky if the consumer cannot keep up.

---

## 9. Paused mode

In paused mode, you pull data manually.

Readable streams support `read()`.

Conceptually:

```text
consumer asks
   |
   v
read()
   |
   v
chunk
```

This gives more explicit control.

---

## 10. pause() and resume()

```js
stream.pause();

stream.resume();
```

Useful when temporarily slowing consumption.

---

## 11. highWaterMark

Readable streams use an internal buffering threshold called `highWaterMark`.

Example:

```js
const stream =
  fs.createReadStream(
    "data.txt",
    {
      highWaterMark: 64 * 1024,
    }
  );
```

This is not a strict maximum memory limit.

It is a buffering threshold used by stream internals.

---

## 12. Reading large files

```js
const stream =
  fs.createReadStream(
    "huge.csv",
    {
      encoding: "utf8",
    }
  );

stream.on(
  "data",
  (chunk) => {
    processChunk(chunk);
  }
);
```

Be careful when chunk boundaries matter.

A logical record may be split across chunks.

---

## 13. Chunk boundaries are arbitrary

Suppose file contains:

```text
hello world
```

You might receive:

```text
"hello "
"world"
```

or:

```text
"hel"
"lo wor"
"ld"
```

Never assume each chunk equals one line/message/object.

This is a very important production concept.

---

## 14. Parsing lines safely

For line-based processing, maintain leftover data between chunks.

Conceptually:

```text
chunk 1: "hello\nwor"
leftover: "wor"

chunk 2: "ld\nnext"
combine:
"world\nnext"
```

This is why parsers need state.

---

## 15. Async iteration

Modern Node.js streams can often be consumed with:

```js
for await (
  const chunk of stream
) {
  console.log(chunk);
}
```

This provides readable async control flow.

---

## 16. Why async iteration is useful

It allows:

```js
try {
  for await (
    const chunk of stream
  ) {
    await processChunk(chunk);
  }
} catch (error) {
  console.error(error);
}
```

This can be easier to reason about than event handlers.

---

## 17. Custom Readable stream

Conceptual example:

```js
import {
  Readable,
} from "node:stream";

const stream =
  new Readable({
    read() {
      this.push("Hello");
      this.push(" World");
      this.push(null);
    },
  });
```

`push(null)` signals end-of-stream.

---

## 18. Real-world uses

Readable streams appear in:

- incoming HTTP bodies
- file downloads
- DB exports
- S3/object storage reads
- logs
- media delivery
- compressed input

---

## 19. Common mistakes

### Mistake 1
Assuming chunks match application records.

### Mistake 2
Ignoring errors.

### Mistake 3
Using data events without understanding flow rate.

### Mistake 4
Loading all chunks into an array and defeating streaming benefits.

### Mistake 5
Treating highWaterMark as a hard max-memory guarantee.

---

## 20. Interview questions

### What is a Readable stream?

A stream that produces data incrementally for consumers.

### What does push(null) mean?

It signals that no more data will be produced.

### What is highWaterMark?

An internal buffering threshold, not a strict hard memory cap.

### Can chunks split logical records?

Yes.

---

## 21. Strong interview answer

> A Readable stream produces data incrementally. It can operate in flowing mode through data events or be consumed manually or with async iteration. Chunks are arbitrary boundaries and often Buffers, so application records such as lines or JSON messages may span multiple chunks. Readable streams use internal buffering controlled by highWaterMark and support backpressure when connected properly to downstream consumers.

---

## Interview-Ready Summary

```text
Readable
   |
   +--> data
   +--> end
   +--> error
   +--> close

Modes:
   flowing
   paused
   async iteration

Important:
chunk boundaries are arbitrary
```

## Practice Task

Read a large newline-delimited file using async iteration.

Correctly handle lines that may be split across chunks.
