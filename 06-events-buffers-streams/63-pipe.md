# Lesson 63 — pipe()

## Why pipe() matters

`pipe()` is one of the simplest and most powerful Node.js stream APIs.

It connects a Readable stream to a Writable stream.

---

## 1. Basic pipe

```js
readable.pipe(writable);
```

Mental model:

```text
Readable
   |
   v
Writable
```

---

## 2. File copy example

```js
import fs from "node:fs";

const source =
  fs.createReadStream(
    "input.txt"
  );

const destination =
  fs.createWriteStream(
    "output.txt"
  );

source.pipe(destination);
```

This copies data incrementally.

---

## 3. Why pipe is better than manual data forwarding

Manual:

```js
source.on(
  "data",
  (chunk) => {
    destination.write(chunk);
  }
);

source.on(
  "end",
  () => {
    destination.end();
  }
);
```

This looks simple, but you must also handle:
- backpressure
- errors
- cleanup
- lifecycle

`pipe()` handles the basic data flow and backpressure more naturally.

---

## 4. Backpressure support

When destination buffer fills:

```text
Writable write() -> false
      |
      v
Readable pauses
      |
      v
Writable drains
      |
      v
Readable resumes
```

`pipe()` coordinates this automatically.

---

## 5. Chaining transforms

```js
source
  .pipe(transformA)
  .pipe(transformB)
  .pipe(destination);
```

Architecture:

```text
Readable
   |
   v
Transform A
   |
   v
Transform B
   |
   v
Writable
```

---

## 6. Compression example

```js
import fs from "node:fs";
import {
  createGzip,
} from "node:zlib";

fs
  .createReadStream(
    "large.txt"
  )
  .pipe(
    createGzip()
  )
  .pipe(
    fs.createWriteStream(
      "large.txt.gz"
    )
  );
```

This is a streaming compression pipeline.

---

## 7. HTTP response example

```js
const file =
  fs.createReadStream(
    "movie.mp4"
  );

file.pipe(res);
```

The response object is Writable.

This avoids loading the entire file into memory.

---

## 8. Does pipe handle all errors?

No.

This is a very important interview point.

`pipe()` does not provide complete centralized error handling and cleanup across a chain.

Example:

```js
source
  .pipe(transform)
  .pipe(destination);
```

If the transform fails, you need to handle errors properly.

For robust multi-stream composition, use `pipeline()`.

---

## 9. end behavior

By default:

```js
source.pipe(destination);
```

when the source ends, the destination is ended automatically.

You can configure:

```js
source.pipe(
  destination,
  {
    end: false,
  }
);
```

Useful when destination should remain open.

---

## 10. Multiple destinations

A readable stream can pipe to multiple writable streams.

Conceptually:

```text
         /--> destination A
Readable
         \--> destination B
```

Be careful:
- backpressure becomes more complex
- one slow destination can affect behavior
- error handling requires attention

---

## 11. unpipe()

```js
source.unpipe(destination);
```

Stops piping to that destination.

Useful for dynamic stream management.

---

## 12. Common mistakes

### Mistake 1
Assuming pipe handles all errors automatically.

It does not.

### Mistake 2
Ignoring source/destination errors.

### Mistake 3
Using readFile + writeFile when streaming would be more memory-efficient.

### Mistake 4
Piping huge data without considering downstream performance.

---

## 13. Interview questions

### What does pipe() do?

It connects a Readable stream to a Writable stream and forwards data while managing basic backpressure.

### Does pipe() handle errors across the whole chain?

No.

### Why use pipeline() instead in production?

Because it provides coordinated error handling and cleanup.

---

## 14. Strong interview answer

> pipe() connects a Readable stream to a Writable stream and automatically forwards chunks while coordinating backpressure. It can also chain Transform streams. However, pipe() does not provide robust centralized error propagation and cleanup across an entire stream chain, so pipeline() is generally safer for production stream workflows.

---

## Interview-Ready Summary

```text
pipe()
   |
   +--> Readable -> Writable
   +--> automatic flow
   +--> backpressure support
   +--> transform chaining

Limitation:
error/cleanup handling is incomplete
```

## Practice Task

Build:

```text
input.txt
   |
   v
uppercase Transform
   |
   v
output.txt
```

using only `pipe()`.
