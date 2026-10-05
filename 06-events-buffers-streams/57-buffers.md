# Lesson 57 — Buffers

## Why Buffers matter

Backend systems do not only process strings and JSON.

They also process raw bytes:

- images
- PDFs
- network packets
- encrypted data
- file chunks
- compressed payloads
- WebSocket frames

Node.js uses `Buffer` to work efficiently with binary data.

---

## 1. What is a Buffer?

A Buffer represents a fixed-size sequence of bytes.

Example:

```js
const buffer =
  Buffer.from("Hello");

console.log(buffer);
```

The string is encoded into bytes.

---

## 2. Text to bytes

```text
"Hello"
   |
   v
UTF-8 encoding
   |
   v
72 101 108 108 111
   |
   v
Buffer
```

---

## 3. Convert Buffer back to string

```js
const buffer =
  Buffer.from("Hello");

console.log(
  buffer.toString("utf8")
);
```

Output:

```text
Hello
```

---

## 4. Buffer.from()

From string:

```js
const buffer =
  Buffer.from(
    "Node.js",
    "utf8"
  );
```

From byte array:

```js
const buffer =
  Buffer.from([
    72,
    105,
  ]);

console.log(buffer.toString());
```

Output:

```text
Hi
```

---

## 5. Buffer.alloc()

Allocates a buffer filled with zeros.

```js
const buffer =
  Buffer.alloc(10);
```

This is safe for general use.

---

## 6. Buffer.allocUnsafe()

```js
const buffer =
  Buffer.allocUnsafe(10);
```

It may be faster because memory is not guaranteed to be zero-filled.

But:

> You must fully overwrite it before reading or exposing its contents.

Otherwise stale memory contents could theoretically leak.

Use only when you understand the trade-off.

---

## 7. Buffer length means bytes

```js
const buffer =
  Buffer.from("hello");

console.log(buffer.length);
```

Output:

```text
5
```

But characters and bytes are not always equal.

Example:

```js
const text = "₹";

console.log(text.length);

console.log(
  Buffer.byteLength(
    text,
    "utf8"
  )
);
```

This is important for:
- API body limits
- file sizes
- protocol payloads
- storage limits

---

## 8. Common encodings

Important encodings:

- utf8
- hex
- base64

---

## 9. Base64

Encode:

```js
const encoded =
  Buffer
    .from("hello")
    .toString("base64");
```

Decode:

```js
const decoded =
  Buffer
    .from(
      encoded,
      "base64"
    )
    .toString("utf8");
```

Important:

```text
Base64 != encryption
```

Anyone can decode Base64.

---

## 10. Hex

```js
const hex =
  Buffer
    .from("hello")
    .toString("hex");
```

Hex is common with:

- hashes
- signatures
- cryptographic keys
- binary identifiers

---

## 11. Buffer.concat()

```js
const a =
  Buffer.from("Hello ");

const b =
  Buffer.from("World");

const result =
  Buffer.concat([a, b]);

console.log(
  result.toString()
);
```

Output:

```text
Hello World
```

---

## 12. Buffer.slice / subarray

Buffers can expose portions of the same underlying memory.

Prefer understanding `subarray()` semantics clearly.

```js
const buffer =
  Buffer.from("abcdef");

const part =
  buffer.subarray(1, 4);

console.log(
  part.toString()
);
```

Output:

```text
bcd
```

---

## 13. Mutability

Buffers are mutable.

```js
const buffer =
  Buffer.from([1, 2, 3]);

buffer[0] = 9;
```

Now:

```js
console.log(buffer);
```

represents:

```text
9, 2, 3
```

---

## 14. Buffers and streams

Streams commonly emit Buffer chunks.

```js
readStream.on(
  "data",
  (chunk) => {
    console.log(
      Buffer.isBuffer(chunk)
    );
  }
);
```

This is one of the most important relationships in this section.

---

## 15. Buffers and HTTP

Incoming request bodies may arrive as chunks of bytes.

Conceptually:

```text
Network
   |
   v
byte chunks
   |
   v
Buffers
   |
   v
combine / decode / parse
   |
   v
JSON / text / file
```

---

## 16. Buffers and crypto

Cryptographic functions frequently work with Buffers.

Example:

```js
const digest =
  crypto
    .createHash("sha256")
    .update("data")
    .digest();

console.log(
  Buffer.isBuffer(digest)
);
```

---

## 17. Buffer vs string

Use strings for text.

Use Buffers for raw bytes.

```text
String
  -> characters / text

Buffer
  -> bytes / binary data
```

---

## 18. Memory considerations

Buffers use memory outside ordinary JavaScript string representations and can contribute significantly to process memory usage.

Large buffers can cause:

- memory pressure
- GC pressure indirectly
- process instability

For large data, prefer streaming rather than loading everything into one Buffer.

---

## 19. Common mistakes

### Mistake 1
Using Base64 as security.

### Mistake 2
Loading huge files into one Buffer.

### Mistake 3
Using allocUnsafe without overwriting memory.

### Mistake 4
Assuming string length equals byte length.

### Mistake 5
Converting binary data to strings unnecessarily.

---

## 20. Interview questions

### What is a Buffer?

A Node.js object representing raw binary data as a sequence of bytes.

### Why does Node.js need Buffers?

Because files, sockets, crypto, compression, and streams operate on bytes rather than only JavaScript strings.

### Is Base64 encryption?

No.

### Difference between Buffer.alloc and Buffer.allocUnsafe?

`alloc` zero-fills memory; `allocUnsafe` skips guaranteed initialization and must be fully overwritten before reading.

---

## 21. Strong interview answer

> A Buffer is Node.js's byte-oriented binary data type. It is used for files, network packets, crypto, compression, and stream chunks. Buffers can encode/decode text using UTF-8, hex, or Base64, but Base64 is only encoding. For large data, streams are usually preferable to building one huge Buffer in memory.

---

## Interview-Ready Summary

```text
Buffer
   |
   +--> raw bytes
   +--> files
   +--> networking
   +--> crypto
   +--> streams
   +--> binary protocols

Key APIs:
Buffer.from()
Buffer.alloc()
Buffer.concat()
Buffer.byteLength()
```

## Practice Task

Create a script that:

1. converts text to Buffer
2. prints UTF-8 byte length
3. converts to Base64
4. decodes back
5. concatenates two Buffers
6. extracts a subarray
