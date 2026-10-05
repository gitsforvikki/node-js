# Lesson 58 — Streams Fundamentals

## Why streams matter

Streams are one of Node.js's most important production concepts.

They allow data to be processed **piece by piece** instead of loading everything into memory first.

Streams are essential for:

- large files
- video/audio delivery
- uploads
- downloads
- compression
- HTTP bodies
- logs
- data pipelines

---

## 1. Without streams

Suppose you read a 5 GB file using:

```js
const data =
  await readFile("huge.zip");
```

The application may attempt to load the entire file into memory.

That is expensive and potentially dangerous.

---

## 2. With streams

```js
const stream =
  fs.createReadStream(
    "huge.zip"
  );
```

The file is processed in smaller chunks.

Mental model:

```text
Huge File

[chunk][chunk][chunk][chunk][chunk]
   |      |      |      |      |
   v      v      v      v      v
application processes incrementally
```

---

## 3. Main stream types

Node.js has four important stream categories:

```text
Readable
Writable
Duplex
Transform
```

---

## 4. Readable stream

Data comes **out** of it.

Examples:

- file read stream
- HTTP request body
- process.stdin

```text
Readable
   |
   v
consumer
```

---

## 5. Writable stream

Data goes **into** it.

Examples:

- file write stream
- HTTP response
- process.stdout

```text
producer
   |
   v
Writable
```

---

## 6. Duplex stream

Can both read and write.

Examples:

- TCP socket

```text
read <----> write
```

---

## 7. Transform stream

A special Duplex stream where output is derived from input.

Examples:

- gzip compression
- encryption
- parsing/transformation

```text
input
  |
  v
transform
  |
  v
output
```

---

## 8. Streams are EventEmitters

Streams inherit EventEmitter behavior.

Readable streams may emit:

- data
- end
- error
- close

Example:

```js
const stream =
  fs.createReadStream(
    "data.txt"
  );

stream.on(
  "data",
  (chunk) => {
    console.log(chunk);
  }
);

stream.on(
  "end",
  () => {
    console.log("done");
  }
);
```

---

## 9. Chunks

A stream delivers data in chunks.

A chunk may be:

- Buffer
- string
- object in object mode

Example:

```js
stream.on(
  "data",
  (chunk) => {
    console.log(
      chunk.length
    );
  }
);
```

---

## 10. Why streams improve memory usage

Without stream:

```text
5 GB file
   |
   v
5 GB memory pressure
```

With stream:

```text
5 GB file
   |
   v
64 KB chunk
   |
   v
process
   |
   v
next chunk
```

Exact chunk sizes depend on stream implementation and configuration, but the concept remains the same.

---

## 11. Backpressure

Backpressure occurs when the producer generates data faster than the consumer can handle it.

Example:

```text
Readable
   | fast
   v
buffer buffer buffer buffer
                     |
                     v
                 Writable
                  slow
```

Without control, memory can grow.

Node.js streams have built-in mechanisms to manage this.

This is one of the most important production benefits of streams.

---

## 12. pipe()

```js
readStream.pipe(writeStream);
```

This connects a readable source to a writable destination.

Conceptually:

```text
Readable
   |
   v
Writable
```

`pipe()` also helps manage backpressure.

---

## 13. pipeline()

For production code, `pipeline()` is often safer because it handles errors and cleanup across multiple streams.

Conceptual:

```js
pipeline(
  source,
  transform,
  destination,
  callback
);
```

Later lessons cover this in depth.

---

## 14. Real-world example

Serve a large file:

Bad:

```js
const file =
  await readFile("movie.mp4");

res.end(file);
```

Better:

```js
const stream =
  fs.createReadStream(
    "movie.mp4"
  );

stream.pipe(res);
```

Now the server does not need to load the entire file before responding.

---

## 15. Object mode

Streams are not limited to bytes.

Object mode allows JavaScript objects as chunks.

Example conceptual pipeline:

```text
Database Row
   |
   v
Transform
   |
   v
JSON Object
```

Useful for:
- ETL
- CSV processing
- logs
- data pipelines

---

## 16. Stream states

Streams have concepts such as:

- flowing mode
- paused mode
- ended/finished
- destroyed

Understanding those becomes important in advanced stream handling.

---

## 17. Common mistakes

### Mistake 1
Using readFile for huge files.

### Mistake 2
Ignoring stream errors.

### Mistake 3
Manually moving data without respecting backpressure.

### Mistake 4
Using `data` event everywhere without understanding flowing mode.

### Mistake 5
Not cleaning up broken pipelines.

---

## 18. Interview questions

### What is a stream in Node.js?

A stream is an abstraction for processing data incrementally over time rather than loading the entire dataset into memory.

### What are the four main stream types?

Readable, Writable, Duplex, Transform.

### Why are streams memory-efficient?

Because they process data in chunks.

### What is backpressure?

A mechanism for slowing the producer when the consumer cannot process data fast enough.

---

## 19. Strong interview answer

> Streams are Node.js abstractions for processing data incrementally. They are useful for large files, HTTP bodies, uploads, compression, and network data because they avoid loading the entire payload into memory. Node.js provides Readable, Writable, Duplex, and Transform streams, and built-in backpressure mechanisms help coordinate producers and consumers.

---

## Interview-Ready Summary

```text
Streams
   |
   +--> Readable
   +--> Writable
   +--> Duplex
   +--> Transform

Benefits:
   memory efficiency
   incremental processing
   backpressure
   composability
```

## Practice Task

Create a 500 MB test file and compare:

1. `fs.readFile`
2. `fs.createReadStream`

Observe memory usage and time-to-first-processing.
