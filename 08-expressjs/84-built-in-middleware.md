# Lesson 84 — Built-in Middleware

## Why built-in middleware matters

Modern Express includes several useful middleware functions out of the box.

The most important ones are:

- `express.json()`
- `express.urlencoded()`
- `express.static()`
- `express.raw()`
- `express.text()`

Understanding when to use each one is more important than simply memorizing their names.

---

## 1. express.json()

Parses JSON request bodies.

```js
app.use(
  express.json()
);
```

Then:

```js
req.body
```

contains the parsed JSON value for matching Content-Type requests.

---

## 2. Add a body limit

Production example:

```js
app.use(
  express.json({
    limit: "1mb",
  })
);
```

Without appropriate limits, clients may send very large request bodies.

---

## 3. express.urlencoded()

Used for:

```text
application/x-www-form-urlencoded
```

Example:

```js
app.use(
  express.urlencoded({
    extended: true,
  })
);
```

This is commonly used with traditional HTML forms.

---

## 4. extended option

Conceptually:

```text
extended: false
  -> simpler key/value parsing

extended: true
  -> richer nested object parsing
```

Use the behavior your application actually needs.

---

## 5. express.text()

Parses matching request bodies as text.

```js
app.use(
  express.text({
    type: "text/plain",
  })
);
```

Then:

```js
typeof req.body
```

is typically:

```text
string
```

---

## 6. express.raw()

Preserves the body as raw bytes.

```js
app.use(
  "/webhooks/payment",
  express.raw({
    type: "application/json",
  })
);
```

Then:

```js
Buffer.isBuffer(
  req.body
);
```

can be true.

This is especially important for webhook signature verification.

---

## 7. Middleware order for raw webhooks

Suppose:

```js
app.use(
  express.json()
);

app.post(
  "/webhooks/payment",
  express.raw({
    type:
      "application/json",
  }),
  webhookHandler
);
```

The global JSON parser may already consume the body before the raw parser sees it.

A safer structure may place the raw webhook route before the global JSON parser:

```js
app.post(
  "/webhooks/payment",
  express.raw({
    type:
      "application/json",
  }),
  webhookHandler
);

app.use(
  express.json()
);
```

Exact setup depends on your provider and Express version/configuration.

---

## 8. express.static()

`express.static()` is a **built-in Express middleware that serves public files directly to the browser or frontend**. Static assets include images (`.jpeg`, `.png`), CSS, JavaScript, HTML, and public PDFs.

### Why do we need it?

Suppose you have 100 images. Without static middleware, you would have to manually create file-serving routes. Instead, Express can look for requested files inside one public folder.

### Step 1 — Create this project structure

```text
express-practice/
├── package.json
├── server.js
└── public/
    ├── cat.jpeg
    ├── logo.png
    └── style.css
```

### Step 2 — Register the static middleware

```js
// server.js (ES modules)
import express from "express";
import path from "node:path";

const app = express();

const publicPath = path.join(process.cwd(), "public");

app.use("/public", express.static(publicPath));

app.get("/", (req, res) => {
  res.send("Express server running");
});

app.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});
```

Start the server **from the project root**, so `process.cwd()` points to `express-practice/`.

### Step 3 — Open the file in a browser

For `public/cat.jpeg`, visit:

```text
http://localhost:3000/public/cat.jpeg
```

Mapping:

```text
Browser: GET /public/cat.jpeg
               |
               v
app.use("/public", express.static(publicPath))
               |
               v
<project-root>/public/cat.jpeg
               |
               v
File exists? → 200 + image/jpeg + image bytes
```

Here `"/public"` is the **URL prefix**, while `publicPath` is the **folder on disk**. The URL prefix does not automatically become another folder inside `publicPath`.

### With vs without a URL prefix

```js
app.use("/public", express.static(publicPath));
// URL: http://localhost:3000/public/cat.jpeg
```

```js
app.use(express.static(publicPath));
// URL: http://localhost:3000/cat.jpeg
```

Both serve the **same physical file**, `public/cat.jpeg`.

### Why might `/public/cat.jpeg` return 404?

Check the file's actual location and name first:

```js
import fs from "node:fs";
import path from "node:path";

const publicPath = path.join(process.cwd(), "public");

console.log("Working directory:", process.cwd());
console.log("Public folder:", publicPath);
console.log(
  "cat.jpeg exists:",
  fs.existsSync(path.join(publicPath, "cat.jpeg"))
);
```

If `existsSync(...)` prints `false`, either the working directory is wrong, the file is in another location, or the filename differs. On Linux, `cat.jpeg`, `cat.jpg`, and `Cat.jpeg` are different names.

Also check **middleware order**:

```js
// Correct: static middleware BEFORE the catch-all 404 handler.
app.use("/public", express.static(publicPath));

app.use("/api/users", userRouter);

app.use((req, res) => {
  res.status(404).json({ message: "Route not found" });
});
```

If your catch-all 404 handler comes first and sends a response, Express never reaches the static middleware. By default, if a static file is missing, `express.static()` passes control onward, where a later handler may return 404.

You can test the response headers with:

```bash
curl -I http://localhost:3000/public/cat.jpeg
```

For an existing JPEG, you should normally see `200 OK` and `Content-Type: image/jpeg`.

### Real-world use: profile images

Suppose `public/uploads/vikash.jpg` exists:

```js
app.use("/uploads", express.static(path.join(process.cwd(), "public", "uploads")));
```

Your API can return an image URL such as `http://localhost:3000/uploads/vikash.jpg`, which a React frontend can display using `<img src={user.profileImage} alt={user.name} />`.

**Remember:** `express.static()` **serves existing files**, but does **not upload** files. For uploads, use a suitable multipart parser/upload workflow. Never expose private documents or sensitive files through a public static folder.

---

## 9. Use absolute paths carefully

Example:

```js
import path from "node:path";

app.use(
  "/public",
  express.static(
    path.join(
      process.cwd(),
      "public"
    )
  )
);
```

This can be more predictable than relying on an unexpected working directory.

---

## 10. Static caching

Static middleware supports caching-related options.

For production static assets, consider:
- max age
- immutable assets
- CDN
- fingerprinted filenames

Do not blindly cache frequently changing files for long periods.

---

## 11. Built-in middleware is still middleware

Example:

```js
express.json()
```

returns a middleware function.

That means it follows the same pipeline behavior:

```text
Request
   |
   v
express.json()
   |
   v
req.body populated
   |
   v
next middleware
```

---

## 12. Route-scoped parser

You do not always need a parser globally.

Example:

```js
app.post(
  "/import",
  express.text({
    type: "text/csv",
  }),
  importCsv
);
```

This keeps parsing behavior narrow.

---

## 13. Parsing based on Content-Type

Body parsing middleware generally uses Content-Type matching.

This is why client headers matter.

Bad client:

```http
Content-Type: text/plain
```

while body contains JSON.

The JSON parser may not parse it as expected.

---

## 14. Built-in middleware is not validation

Again:

```text
express.json
  -> parse

Zod/Joi/custom validation
  -> validate
```

Never treat a successfully parsed body as trustworthy.

---

## 15. Built-in middleware is not file-upload middleware

`multipart/form-data` requires a multipart parser.

`express.json()` does not handle file uploads.

---

## 16. Common mistakes

### Mistake 1
No parser body limits.

### Mistake 2
Global JSON parser breaking signed webhook raw-body verification.

### Mistake 3
Using express.json for multipart.

### Mistake 4
Treating parsing as validation.

### Mistake 5
Serving sensitive directories with express.static.

---

## 17. Interview questions

### What are common built-in Express middleware functions?

`express.json`, `express.urlencoded`, `express.text`, `express.raw`, and `express.static`.

### What is express.raw used for?

To preserve the request body as raw bytes, often for signature verification.

### Does express.json validate the body?

No.

### Can express.json handle multipart file uploads?

No.

---

## 18. Strong interview answer

> Express includes built-in middleware for common parsing and static-serving concerns. I use express.json for JSON APIs with a body limit, express.urlencoded for form-encoded bodies, express.raw when exact bytes are required such as signed webhooks, express.text for text payloads, and express.static for public assets. These middleware functions parse or serve data but do not perform business validation.

---

## Interview-Ready Summary

```text
express.json()
  -> JSON

express.urlencoded()
  -> form encoded

express.text()
  -> text

express.raw()
  -> Buffer/raw bytes

express.static()
  -> files

Important:
limits
Content-Type
middleware order
raw webhook handling
```

## Practice Task

Configure an app with:

1. raw payment webhook route
2. JSON API routes
3. URL-encoded form route
4. static public assets
5. strict body-size limits
