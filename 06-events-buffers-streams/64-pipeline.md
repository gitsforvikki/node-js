# Lesson 64 — pipeline()

## Why pipeline() matters

`pipeline()` is the production-friendly way to connect streams.

It solves one of the biggest problems with plain `pipe()` chains:

> coordinated error handling and cleanup.

---

## 0. First understand the problem: why do we need pipe or pipeline?

Imagine copying a 1 GB file. With `readFile()`, Node.js attempts to load the entire file into memory:

```js
import fs from "node:fs/promises";

const data = await fs.readFile("large-file.txt");
await fs.writeFile("output.txt", data);
```

For large files, holding the full content in memory can be expensive. **Streams** read and write in chunks instead.

### Option A — `pipe()`

```js
import fs from "node:fs";

const readable = fs.createReadStream("large-file.txt");
const writable = fs.createWriteStream("output.txt");

readable.pipe(writable);
```

Flow:

```text
large-file.txt  ->  Readable Stream  --pipe()-->  Writable Stream  ->  output.txt
                      (chunks)                       (chunks)
```

How it works:

1. `createReadStream()` reads chunks from the source file.
2. `pipe()` sends those chunks to the writable stream.
3. `createWriteStream()` writes them to the destination file.
4. The destination normally finishes when the source ends.

`pipe()` also handles **backpressure**: if writing is slower than reading, it pauses/resumes the readable stream as needed.

**Limitation:** `pipe()` does not coordinate all errors and destruction across the chain. For example, if `large-file.txt` does not exist, the read stream emits an `error` event; `pipe()` alone does not provide a complete error boundary.

### Option B — `pipeline()` (recommended for complete jobs)

```js
import fs from "node:fs";
import { pipeline } from "node:stream/promises";

async function copyFile() {
  try {
    await pipeline(
      fs.createReadStream("large-file.txt"),
      fs.createWriteStream("output.txt")
    );
    console.log("File copied successfully");
  } catch (error) {
    console.error("File copy failed:", error.message);
  }
}

copyFile();
```

`pipeline()` also transfers chunks and handles backpressure, but additionally:

1. Connects the stream chain.
2. Propagates errors to one callback or Promise rejection.
3. Destroys relevant streams on failure (with some documented exceptions).
4. Waits until the pipeline completes; the Promise version can be `await`ed.

If `large-file.txt` is missing, the `catch` block runs instead of leaving you to wire every stream's error handling separately. A failed copy may leave a partial `output.txt`; remove it if your application requires all-or-nothing output.

---

## Comparison — pipe() vs pipeline()

| Feature | `pipe()` | `pipeline()` |
| --- | --- | --- |
| Purpose | Connect readable to writable | Manage a complete stream chain |
| Backpressure | Yes | Yes |
| Multiple steps | Chain `.pipe()` calls | Pass multiple streams as arguments |
| Error coordination | Manual | Built in |
| Cleanup after error | Largely manual | Coordinated stream destruction |
| Completion with async/await | Not directly | Yes, `node:stream/promises` |
| Best fit | Manually controlled/simple connections | File processing, compression, production workflows |

**Important:** `pipe()` is a method on a readable stream (`readable.pipe(writable)`). `pipeline()` is a utility imported from `node:stream` (callback) or `node:stream/promises` (Promise).

---

## Practical example — compress a log file

This example uses three types of streams:

```text
app.log -> Readable -> Transform (gzip) -> Writable -> app.log.gz
```

### With pipe()

```js
import fs from "node:fs";
import zlib from "node:zlib";

fs.createReadStream("app.log")
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream("app.log.gz"));
```

It works on success, but a production version needs proper error handling and cleanup.

### With pipeline()

```js
import fs from "node:fs";
import zlib from "node:zlib";
import { pipeline } from "node:stream/promises";

async function compressLog() {
  try {
    await pipeline(
      fs.createReadStream("app.log"),
      zlib.createGzip(),
      fs.createWriteStream("app.log.gz")
    );

    console.log("Compression completed");
  } catch (error) {
    console.error("Compression failed:", error.message);
  }
}

compressLog();
```

The file is processed in chunks; `pipeline()` **does not load the entire file into memory**.

### When should I use each?

**Use `pipe()`** when you want a simple connection and plan to manage errors/lifecycle yourself, or need direct control such as `unpipe()`.

**Use `pipeline()`** for tasks such as copying large files, compressing uploads/logs, processing CSV files, or connecting multiple transforms where errors and cleanup matter.

**Interview answer:** `pipe()` connects streams and handles data flow/backpressure. `pipeline()` connects streams **and** coordinates completion, errors, and cleanup. Its Promise API supports `async/await`, so it's generally the safer choice for a complete production stream job.

**Quick test:** What happens if `input.txt` doesn't exist while you run `fs.createReadStream("input.txt").pipe(fs.createWriteStream("output.txt"))`? You need an error listener or coordinated handling; plain `pipe()` does not automatically handle that error for you.

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
