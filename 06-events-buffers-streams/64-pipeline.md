# Lesson 64 — pipeline()

## Why pipeline() matters

`pipeline()` is the production-friendly way to connect streams.

It solves one of the biggest problems with plain `pipe()` chains:

> coordinated error handling and cleanup.

---

## 1. Basic pipeline

Callback style:

```js
import {
  pipeline,
} from "node:stream";

pipeline(
  source,
  transform,
  destination,
  (error) => {
    if (error) {
      console.error(
        "Pipeline failed",
        error
      );
      return;
    }

    console.log(
      "Pipeline complete"
    );
  }
);
```

---

## 2. Promise-based pipeline

Modern Node.js also provides:

```js
import {
  pipeline,
} from "node:stream/promises";
```

Usage:

```js
await pipeline(
  source,
  transform,
  destination
);
```

This works naturally with async/await.

---

## 3. Compression example

```js
import fs from "node:fs";
import {
  createGzip,
} from "node:zlib";
import {
  pipeline,
} from "node:stream/promises";

await pipeline(
  fs.createReadStream(
    "large.txt"
  ),
  createGzip(),
  fs.createWriteStream(
    "large.txt.gz"
  )
);
```

---

## 4. Why pipeline is safer

Suppose:

```text
source
  |
  v
transform
  |
  v
destination
```

If transform fails, pipeline coordinates stream destruction and error propagation.

With plain pipe chains, you would need more manual handling.

---

## 5. Error propagation

```js
try {
  await pipeline(
    source,
    transform,
    destination
  );
} catch (error) {
  console.error(
    "Pipeline failed:",
    error
  );
}
```

This creates one error boundary for the whole flow.

---

## 6. Cleanup

Broken stream chains can leak:

- file descriptors
- sockets
- memory
- resources

`pipeline()` helps destroy connected streams when failures occur.

This is one of its most important production benefits.

---

## 7. pipeline with async generators

Modern stream pipelines can also work with async iterables/generators in supported patterns.

Conceptually:

```text
Readable
   |
   v
async generator
   |
   v
Writable
```

This can make transformation logic very expressive.

---

## 8. Abort/cancellation

Production stream flows may need cancellation.

A common modern pattern uses `AbortSignal` where supported.

Conceptually:

```text
request cancelled
    |
    v
AbortController
    |
    v
pipeline stops
```

This is useful for:
- HTTP client disconnects
- timeouts
- user cancellation

---

## 9. pipeline vs pipe

```text
pipe()
  simple
  backpressure support
  manual error handling

pipeline()
  backpressure support
  coordinated errors
  cleanup
  Promise API
  production safer
```

---

## 10. Real backend example

Large CSV import:

```text
CSV File
   |
   v
Read Stream
   |
   v
CSV Parser
   |
   v
Validation Transform
   |
   v
Database Writer
```

Using `pipeline()` gives one lifecycle boundary for the full processing chain.

---

## 11. Common mistakes

### Mistake 1
Building long pipe chains without error coordination.

### Mistake 2
Ignoring pipeline rejection.

### Mistake 3
Not closing external resources used inside custom transforms.

### Mistake 4
Assuming pipeline makes CPU-heavy transformation safe.

CPU-heavy transformation can still block the event loop.

---

## 12. Interview questions

### What is pipeline()?

A Node.js utility for composing streams with coordinated error propagation and cleanup.

### Why is it safer than pipe()?

Because it manages failures across the entire chain and destroys related streams when necessary.

### Is there an async/await version?

Yes, through `node:stream/promises`.

---

## 13. Strong interview answer

> pipeline() composes multiple streams while managing backpressure, errors, and cleanup across the entire chain. Compared with manually chaining pipe(), pipeline provides a single completion/error boundary and helps destroy connected streams if one fails. The promise-based API integrates well with async/await and is generally preferred for production stream pipelines.

---

## Interview-Ready Summary

```text
pipeline()
   |
   +--> compose streams
   +--> backpressure
   +--> centralized errors
   +--> cleanup
   +--> async/await support

Preferred for:
production pipelines
```

## Practice Task

Create a pipeline that:

1. reads a text file
2. transforms text to uppercase
3. compresses with gzip
4. writes the compressed output
5. catches pipeline errors
