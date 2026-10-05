# Lesson 60 — Writable Streams

## What is a Writable stream?

A Writable stream represents a destination that consumes data incrementally.

Examples:

- `fs.createWriteStream()`
- HTTP response objects
- `process.stdout`
- sockets
- custom writable streams

---

## 1. Basic file write stream

```js
import fs from "node:fs";

const stream =
  fs.createWriteStream(
    "output.txt"
  );

stream.write("Hello\n");
stream.write("World\n");

stream.end();
```

---

## 2. write()

```js
stream.write("data");
```

This writes a chunk to the stream.

Chunks may be:

- strings
- Buffers
- objects in object mode

---

## 3. end()

```js
stream.end();
```

Signals:

> No more data will be written.

You can also write final data:

```js
stream.end("final chunk");
```

---

## 4. finish event

```js
stream.on(
  "finish",
  () => {
    console.log(
      "All data flushed"
    );
  }
);
```

`finish` means the writable side has processed all submitted data.

---

## 5. error event

```js
stream.on(
  "error",
  (error) => {
    console.error(error);
  }
);
```

Always handle stream failures.

---

## 6. write() return value

This is extremely important.

```js
const canContinue =
  stream.write(chunk);
```

If it returns:

```text
true
```

the internal buffer can accept more data.

If it returns:

```text
false
```

the producer should slow down and wait for `drain`.

---

## 7. Backpressure with writable streams

Example:

```js
if (!stream.write(chunk)) {
  await once(
    stream,
    "drain"
  );
}
```

Conceptually:

```text
Producer fast
    |
    v
Writable buffer fills
    |
    v
write() -> false
    |
    v
producer pauses
    |
    v
drain
    |
    v
producer resumes
```

This is the core of backpressure.

---

## 8. Why ignoring write() is dangerous

Bad:

```js
for (
  const chunk of hugeData
) {
  stream.write(chunk);
}
```

If the destination is slower than the producer, internal buffering may grow dramatically.

Possible result:

- high memory usage
- GC pressure
- process instability

---

## 9. Practical backpressure example

```js
import {
  once,
} from "node:events";

for (
  const chunk of chunks
) {
  const ok =
    stream.write(chunk);

  if (!ok) {
    await once(
      stream,
      "drain"
    );
  }
}

stream.end();
```

---

## 10. HTTP response is Writable

In:

```js
http.createServer(
  (req, res) => {
    res.write("hello");
    res.end();
  }
);
```

`res` behaves as a Writable stream.

That means large responses should respect streaming concepts.

---

## 11. process.stdout

```js
process.stdout.write(
  "Hello\n"
);
```

`stdout` is also stream-based.

---

## 12. Custom Writable stream

```js
import {
  Writable,
} from "node:stream";

const writable =
  new Writable({
    write(
      chunk,
      encoding,
      callback
    ) {
      console.log(
        chunk.toString()
      );

      callback();
    },
  });
```

The callback signals that the chunk has been processed.

---

## 13. Object mode

Writable streams can accept objects.

```js
const writable =
  new Writable({
    objectMode: true,

    write(
      object,
      encoding,
      callback
    ) {
      console.log(object);

      callback();
    },
  });
```

Useful for:
- data pipelines
- database records
- structured transformations

---

## 14. highWaterMark

Writable streams also use `highWaterMark`.

It influences when `write()` starts returning false.

It is a buffering threshold, not a hard memory cap.

---

## 15. cork() and uncork()

Writable streams support batching small writes.

```js
stream.cork();

stream.write("A");
stream.write("B");

stream.uncork();
```

Useful in specific high-throughput scenarios.

Do not use blindly.

---

## 16. finish vs close

```text
finish
  -> all writes processed

close
  -> underlying resource closed
```

They represent different lifecycle events.

---

## 17. Real-world examples

Writable streams are used for:

- file uploads
- writing exports
- HTTP responses
- logs
- sockets
- compression pipelines
- cloud storage uploads

---

## 18. Common mistakes

### Mistake 1
Ignoring write() return value.

### Mistake 2
Not waiting for drain.

### Mistake 3
Calling write after end.

### Mistake 4
Ignoring error events.

### Mistake 5
Creating massive buffers before writing and losing streaming benefits.

---

## 19. Interview questions

### What is a Writable stream?

A stream that accepts data incrementally.

### What does write() returning false mean?

The internal buffer has reached its threshold and the producer should pause until the drain event.

### What does finish mean?

All submitted data has been processed by the writable stream.

### Why is drain important?

It signals that buffered data has been flushed enough for writing to resume.

---

## 20. Strong interview answer

> A Writable stream consumes data incrementally. Its write() method returns a boolean indicating whether more data should be written immediately. If write() returns false, the producer should pause and wait for the drain event. This backpressure mechanism prevents fast producers from overwhelming slow destinations and causing unbounded memory growth.

---

## Interview-Ready Summary

```text
Writable
   |
   +--> write()
   +--> end()
   +--> finish
   +--> error
   +--> drain

Backpressure:
write() false
   |
   v
wait drain
   |
   v
resume
```

## Section 6 Progress Map

```text
Events
   |
   +--> EventEmitter
   +--> custom events

Binary Data
   |
   +--> Buffers

Streams
   |
   +--> fundamentals
   +--> Readable
   +--> Writable
   |
   +--> next:
          Duplex
          Transform
          pipe
          pipeline
          backpressure
          large-file processing
```

## Practice Task

Create a custom Writable stream that:

1. accepts JSON objects in object mode
2. converts each object to JSON text
3. writes the output to a file
4. respects backpressure
5. logs the finish event
