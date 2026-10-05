# Lesson 62 — Transform Streams

## Why Transform streams matter

Transform streams are used when data should be processed **while it flows**.

Examples:

- compression
- encryption
- parsing
- formatting
- filtering
- CSV conversion
- log transformation

They are one of the most useful stream types in production Node.js systems.

---

## 1. What is a Transform stream?

A Transform stream is a special Duplex stream where:

> data written to the writable side is processed and emitted through the readable side.

Mental model:

```text
Input
  |
  v
Transform Logic
  |
  v
Output
```

---

## 2. Basic example

```js
import {
  Transform,
} from "node:stream";

const uppercase =
  new Transform({
    transform(
      chunk,
      encoding,
      callback
    ) {
      const output =
        chunk
          .toString()
          .toUpperCase();

      callback(
        null,
        output
      );
    },
  });
```

Usage:

```js
process.stdin
  .pipe(uppercase)
  .pipe(process.stdout);
```

---

## 3. The _transform contract

A Transform implementation receives:

- `chunk`
- `encoding`
- `callback`

Conceptually:

```js
callback(error, transformedChunk);
```

Success:

```js
callback(
  null,
  transformed
);
```

Failure:

```js
callback(error);
```

---

## 4. push() vs callback output

Instead of returning one transformed chunk through callback, you can push data:

```js
transform(
  chunk,
  encoding,
  callback
) {
  this.push(
    chunk.toString().toUpperCase()
  );

  callback();
}
```

Useful when one input chunk creates multiple output chunks.

---

## 5. One input can create many outputs

Example:

```text
Input:
"a,b,c"

Output chunks:
"a"
"b"
"c"
```

Transform streams are not required to preserve one-to-one chunk mapping.

---

## 6. One output can depend on many inputs

A parser may need to buffer data internally.

Example:

```text
chunk 1:
"{\"name\":"

chunk 2:
"\"Vikash\"}"

combined:
{"name":"Vikash"}
```

Transform logic may need to preserve partial state between chunks.

---

## 7. _flush()

Sometimes final buffered data must be emitted when input ends.

Example concept:

```js
flush(callback) {
  if (this.buffer) {
    this.push(this.buffer);
  }

  callback();
}
```

Useful for:
- parsers
- encoders
- line processors

---

## 8. Built-in Transform examples

### gzip

```js
import {
  createGzip,
} from "node:zlib";

const gzip =
  createGzip();
```

Flow:

```text
raw file
   |
   v
gzip Transform
   |
   v
compressed bytes
```

---

## 9. Crypto transforms

Some encryption/decryption APIs can also be used as streams.

Conceptually:

```text
plaintext
   |
   v
cipher transform
   |
   v
ciphertext
```

---

## 10. Object-mode Transform

```js
const transform =
  new Transform({
    objectMode: true,

    transform(
      user,
      encoding,
      callback
    ) {
      callback(
        null,
        {
          ...user,
          fullName:
            user.name.toUpperCase(),
        }
      );
    },
  });
```

Useful for:
- ETL
- records
- CSV parsing
- DB export processing

---

## 11. Backpressure still applies

Because Transform is both Writable and Readable:

```text
upstream
   |
   v
Transform
   |
   v
downstream
```

If downstream is slow, backpressure should propagate upstream when streams are connected properly.

---

## 12. Error handling

Bad:

```js
transform(
  chunk,
  enc,
  callback
) {
  JSON.parse(
    chunk.toString()
  );
}
```

If parsing fails, signal the error:

```js
try {
  const parsed =
    JSON.parse(
      chunk.toString()
    );

  callback(
    null,
    parsed
  );
} catch (error) {
  callback(error);
}
```

---

## 13. Production example — log redaction

Input:

```text
user=1 token=secret123
```

Transform:

```text
user=1 token=[REDACTED]
```

This allows safe streaming processing without loading an entire log file.

---

## 14. Transform vs normal function

Normal function:

```text
whole input
   |
   v
function
   |
   v
whole output
```

Transform stream:

```text
chunk
  |
  v
transform
  |
  v
chunk
  |
  repeat
```

Streams are ideal when data is large or continuous.

---

## 15. Common mistakes

### Mistake 1
Assuming chunks equal records.

### Mistake 2
Forgetting to call callback.

The stream can stall.

### Mistake 3
Doing huge synchronous CPU work inside transform.

This can still block the event loop.

### Mistake 4
Ignoring errors.

### Mistake 5
Holding all chunks internally and defeating streaming.

---

## 16. Interview questions

### What is a Transform stream?

A Duplex stream whose writable input is processed to produce readable output.

### Give examples.

Compression, encryption, parsing, formatting.

### Does one input chunk always produce one output chunk?

No.

### Does backpressure work through Transform streams?

Yes, when connected correctly.

---

## 17. Strong interview answer

> A Transform stream is a specialized Duplex stream where input written to the writable side is transformed into output on the readable side. It is commonly used for compression, encryption, parsing, and format conversion. Transform streams process data incrementally and participate in backpressure, making them efficient for large or continuous data flows.

---

## Interview-Ready Summary

```text
Transform
   |
   +--> Duplex subclass
   +--> input -> processing -> output
   +--> streaming transformation
   +--> backpressure aware

Examples:
gzip
encryption
parser
formatter
```

## Practice Task

Build a Transform stream that:
1. receives text
2. removes extra whitespace
3. converts to uppercase
4. writes the transformed output to a file
