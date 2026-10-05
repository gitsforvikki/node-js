# Lesson 31 — Buffer

## Why Buffers Exist

JavaScript strings are designed for text.

Servers also handle raw binary data:

- images
- PDFs
- encrypted values
- network packets
- compressed data
- file chunks
- WebSocket frames

Node.js uses `Buffer` to represent raw bytes.

## Create a Buffer

```js
const buffer = Buffer.from("Hello");

console.log(buffer);
```

You may see byte values represented in hexadecimal.

## Convert Back to Text

```js
console.log(buffer.toString("utf8"));
```

Output:

```text
Hello
```

## Mental Model

```text
"Hello"
   |
 UTF-8 encoding
   |
   v
bytes
72 101 108 108 111
   |
   v
Buffer
```

## Buffer.from()

From string:

```js
const buf = Buffer.from(
  "Node.js",
  "utf8"
);
```

From array:

```js
const buf = Buffer.from([
  72,
  105,
]);

console.log(buf.toString());
// Hi
```

## Buffer.alloc()

Allocates a zero-filled buffer.

```js
const buf = Buffer.alloc(10);
```

## Buffer.allocUnsafe()

```js
const buf =
  Buffer.allocUnsafe(10);
```

This can be faster because memory is not guaranteed to be initialized.

Never expose or use contents before fully overwriting the buffer.

For normal application code, prefer `Buffer.alloc` unless you clearly understand the performance/security trade-off.

## Buffer Length

```js
const buf = Buffer.from("Hello");

console.log(buf.length);
```

Length is bytes, not always characters.

Example:

```js
const text = "₹";

console.log(text.length);
console.log(
  Buffer.byteLength(text, "utf8")
);
```

Character count and byte count can differ.

This matters for:
- body limits
- database sizes
- protocol limits
- network payloads

## Encodings

Common encodings:
- utf8
- hex
- base64

### Base64

```js
const encoded = Buffer
  .from("secret")
  .toString("base64");

console.log(encoded);
```

Decode:

```js
const decoded = Buffer
  .from(encoded, "base64")
  .toString("utf8");
```

Important:

> Base64 is encoding, not encryption.

It provides no confidentiality.

## Hex

```js
const hex = Buffer
  .from("hello")
  .toString("hex");
```

Common with:
- hashes
- signatures
- crypto
- binary identifiers

## Comparing Buffers

```js
const a = Buffer.from("hello");
const b = Buffer.from("hello");

console.log(a.equals(b));
// true
```

For secret values, timing-safe comparisons may be more appropriate.

## Concatenate Buffers

```js
const a = Buffer.from("Hello ");
const b = Buffer.from("World");

const result = Buffer.concat([a, b]);

console.log(result.toString());
```

## Buffers and Streams

Streams often emit Buffer chunks.

```js
stream.on("data", (chunk) => {
  console.log(
    Buffer.isBuffer(chunk)
  );
});
```

This is how Node handles large binary or text data incrementally.

## Buffers and HTTP

Request bodies may arrive as Buffer chunks.

Conceptually:

```text
Network
   |
   v
byte chunks
   |
   v
Buffer
   |
   v
decode/parse
   |
   v
JSON/text/file
```

## Buffer and TypedArray

Modern Buffer is closely related to JavaScript typed-array infrastructure and represents byte-oriented memory.

This helps Node integrate efficiently with:
- native code
- networking
- filesystem APIs

## Security Considerations

### Never confuse Base64 with encryption

Bad security:

```text
password -> base64 -> "protected"
```

Anyone can decode it.

### Be careful with allocUnsafe

Uninitialized memory can expose stale bytes if not completely overwritten.

### Limit incoming data

An attacker can send huge payloads.

Enforce:
- body-size limits
- file-size limits
- streaming boundaries

## Industry Example

Payment signature:

```js
const expected =
  Buffer.from(expectedSignature);

const received =
  Buffer.from(receivedSignature);
```

File upload:

```text
HTTP request
   |
   v
Buffer chunks
   |
   v
validation/storage
```

## Interview Questions

### What is a Buffer in Node.js?

A Buffer is a Node.js object for working with raw binary data directly as bytes.

### Why are Buffers important in backend development?

Because files, network packets, encryption, compression and many streams operate on binary data rather than JavaScript strings.

### Is Base64 encryption?

No. Base64 is only an encoding format.

## Interview-Ready Summary

```text
Buffer
   = raw bytes

Common uses
   -> files
   -> streams
   -> network data
   -> crypto
   -> encodings

Buffer.from()
Buffer.alloc()
Buffer.concat()
Buffer.byteLength()
```

## Section 3 Final Mental Model

After this section, connect the core modules like this:

```text
Node.js Core Runtime
        |
        +--> path
        |     filesystem paths
        |
        +--> fs / fs/promises
        |     file operations
        |
        +--> os
        |     machine/runtime information
        |
        +--> url
        |     URL parsing
        |
        +--> events
        |     event-driven architecture
        |
        +--> http
        |     servers and requests
        |
        +--> crypto
        |     security primitives
        |
        +--> util
        |     runtime utilities
        |
        +--> Buffer
              binary data
```

These modules are not isolated topics. Together they form the low-level foundation used underneath frameworks, file processing, networking, authentication, streams, uploads, webhooks and production backend systems.

## Practice Task

Build a small Node.js server without Express that:
1. parses request URLs
2. reads a JSON file using `fs/promises`
3. creates a secure request ID using `crypto.randomUUID()`
4. emits a request event with `EventEmitter`
5. returns JSON through `node:http`
6. uses `path` for file resolution
7. returns Buffer-based file content from one route
