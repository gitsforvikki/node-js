# Lesson 74 — Request Body Parsing

## Why body parsing matters

In raw Node.js, there is no automatic:

```js
req.body
```

The HTTP request body arrives as a Readable stream.

You must:
- collect or stream chunks
- enforce size limits
- inspect Content-Type
- decode correctly
- parse safely
- handle malformed data

This is exactly what frameworks hide for you.

---

## 1. Raw request body

```js
const chunks = [];

req.on(
  "data",
  (chunk) => {
    chunks.push(chunk);
  }
);

req.on(
  "end",
  () => {
    const body =
      Buffer.concat(
        chunks
      );

    console.log(body);
  }
);
```

---

## 2. Why the body arrives in chunks

Network data does not necessarily arrive in one piece.

Conceptually:

```text
HTTP body:
{"name":"Vikash"}

may arrive as:

chunk 1:
{"name":

chunk 2:
"Vikash"}
```

Never assume one `data` event contains the full body.

---

## 3. JSON body parsing

```js
function parseJsonBody(
  req
) {
  return new Promise(
    (resolve, reject) => {
      const chunks = [];

      req.on(
        "data",
        (chunk) => {
          chunks.push(chunk);
        }
      );

      req.on(
        "end",
        () => {
          try {
            const raw =
              Buffer
                .concat(chunks)
                .toString("utf8");

            resolve(
              JSON.parse(raw)
            );
          } catch (error) {
            reject(error);
          }
        }
      );

      req.on(
        "error",
        reject
      );
    }
  );
}
```

---

## 4. Empty JSON body

Be careful with:

```js
JSON.parse("");
```

This throws.

Your API should define whether an empty body is:
- allowed
- treated as {}
- rejected

Consistency matters.

---

## 5. Content-Type validation

Before parsing JSON:

```js
const contentType =
  req.headers[
    "content-type"
  ];
```

You should verify it represents JSON.

Example:

```text
application/json
application/json; charset=utf-8
```

Avoid exact-string-only checks when parameters such as charset may be included.

---

## 6. Body size limits

This is critical.

```js
const MAX_BODY_SIZE =
  1024 * 1024;
```

Then:

```js
let total = 0;

req.on(
  "data",
  (chunk) => {
    total +=
      chunk.length;

    if (
      total >
      MAX_BODY_SIZE
    ) {
      req.destroy();
    }
  }
);
```

Why?

Without limits, an attacker may exhaust server memory.

---

## 7. Better parsing flow

```text
Request
   |
   v
check Content-Type
   |
   v
read chunks
   |
   v
enforce size
   |
   v
decode bytes
   |
   v
parse JSON
   |
   v
validate object schema
```

Parsing and validation are separate concerns.

---

## 8. Parsing is not validation

This:

```json
{
  "email": 123,
  "age": "hello"
}
```

is valid JSON.

But it may be invalid application data.

So:

```text
JSON parsing
   !=
business validation
```

After parsing, validate schema.

---

## 9. Form data

Common Content-Types include:

```text
application/json
application/x-www-form-urlencoded
multipart/form-data
text/plain
```

Each needs different parsing behavior.

---

## 10. multipart/form-data

Used commonly for file uploads.

Do not manually parse multipart in production unless you have a strong reason.

Why?

Multipart includes:
- boundaries
- headers per part
- binary payloads
- large streams

Use mature parsers/libraries.

---

## 11. Large file uploads

Do not collect an entire multi-GB file body into memory.

Use streaming.

```text
Incoming request
   |
   v
stream parser
   |
   v
storage destination
```

---

## 12. Raw body and webhook verification

This is a highly important real-world topic.

Some payment/webhook providers sign the exact raw request bytes.

If you parse/re-serialize first:

```text
raw bytes
   |
   v
JSON.parse
   |
   v
JSON.stringify
```

the byte representation may change.

Signature verification may fail.

For signed webhooks, preserve raw body exactly according to provider requirements.

---

## 13. Character encoding

Most JSON APIs use UTF-8.

Convert carefully:

```js
bodyBuffer.toString(
  "utf8"
);
```

Do not assume every binary body should become text.

---

## 14. Error responses

Malformed JSON:

```text
400 Bad Request
```

Unsupported media type:

```text
415 Unsupported Media Type
```

Oversized body:

```text
413 Content Too Large
```

These are more precise than returning 500.

---

## 15. Client disconnect

A client may abort during upload.

Your parser should consider:
- aborted request
- stream errors
- cleanup

Avoid continuing expensive work unnecessarily.

---

## 16. Common mistakes

### Mistake 1
Assuming req.body exists.

### Mistake 2
No body-size limit.

### Mistake 3
Parsing every request as JSON regardless of Content-Type.

### Mistake 4
Treating parsing as validation.

### Mistake 5
Destroying raw webhook body before signature verification.

### Mistake 6
Buffering huge uploads entirely in memory.

---

## 17. Interview questions

### How do you parse a JSON body in raw Node.js?

Read the IncomingMessage stream chunks, enforce limits, combine/decode them, then JSON.parse the result.

### Why enforce body limits?

To protect memory and prevent denial-of-service behavior.

### Is valid JSON necessarily valid API input?

No. It still needs schema/business validation.

### Why might a webhook need raw body?

Because cryptographic signatures may be calculated over the exact original bytes.

---

## 18. Strong interview answer

> In raw Node.js, request bodies arrive as a stream. I inspect Content-Type, collect or stream the chunks, enforce a strict size limit, decode the body, and then parse it. JSON parsing only proves syntactic validity, so I validate the resulting object separately. For large uploads I stream instead of buffering, and for signed webhooks I preserve the exact raw bytes before parsing because signature verification may depend on them.

---

## Interview-Ready Summary

```text
Body Parsing
   |
   +--> request stream
   +--> chunks
   +--> size limit
   +--> Content-Type
   +--> decoding
   +--> parsing
   +--> validation

Important:
raw webhook body
large uploads -> stream
```

## Practice Task

Write `parseJsonBody(req)` that:

1. supports JSON only
2. limits body to 1 MB
3. rejects malformed JSON
4. handles stream errors
5. preserves raw Buffer alongside parsed data
