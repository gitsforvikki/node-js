# Lesson 26 — url

## Why URL Handling Matters

URLs are structured data.

Do not manipulate them with fragile string splitting.

Bad:

```js
const id = req.url.split("?")[0].split("/")[2];
```

Better:

Use standard URL parsing APIs.

## The WHATWG URL API

Modern Node.js supports the standard `URL` class.

```js
const url = new URL(
  "https://example.com/products?id=10&page=2"
);
```

## Important Properties

```js
console.log(url.protocol);
console.log(url.hostname);
console.log(url.port);
console.log(url.pathname);
console.log(url.search);
console.log(url.hash);
console.log(url.origin);
```

## Search Parameters

```js
console.log(url.searchParams.get("id"));
console.log(url.searchParams.get("page"));
```

You can also modify them:

```js
url.searchParams.set("page", "3");
url.searchParams.append("sort", "price");
```

## Iterating Query Params

```js
for (const [key, value] of url.searchParams) {
  console.log(key, value);
}
```

## Relative URLs Need a Base

This fails:

```js
new URL("/users?id=10");
```

because the URL is relative.

Provide a base:

```js
const url = new URL(
  "/users?id=10",
  "https://api.example.com"
);
```

## HTTP Server Example

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  const requestUrl = new URL(
    req.url,
    `http://${req.headers.host}`
  );

  console.log(requestUrl.pathname);
  console.log(requestUrl.searchParams.get("page"));

  res.end("OK");
});
```

## URL Encoding

URLs may contain special characters.

Example:

```js
const url = new URL("https://example.com");

url.searchParams.set("q", "node js & backend");

console.log(url.toString());
```

The API safely encodes the query.

## fileURLToPath

With ES Modules, module URLs are often represented using `import.meta.url`.

```js
import {
  fileURLToPath,
} from "node:url";

const filename = fileURLToPath(import.meta.url);
```

This converts a file URL to a filesystem path.

## URL vs path

Do not confuse them.

```text
URL
  https://example.com/users

Filesystem path
  /home/user/project/file.js
```

Use:
- `node:url` for URLs
- `node:path` for filesystem paths

## Security Considerations

Be careful when accepting arbitrary URLs for server-side fetching.

Attackers may attempt SSRF by requesting:
- localhost
- cloud metadata endpoints
- internal services
- private network ranges

URL parsing is only the first step. You still need authorization and destination validation.

## Interview Questions

### Why use URL instead of manually splitting strings?

Because the URL API correctly understands protocol, hostname, port, pathname, search parameters, encoding and other URL semantics.

### What is URLSearchParams?

A standard API for reading and modifying query-string parameters.

## Summary

```text
URL
  -> parse URL
  -> pathname
  -> hostname
  -> query params
  -> encoding
  -> safe construction
```

## Practice Task

Build a raw Node HTTP server that reads:

```text
/products?page=2&limit=10&sort=price
```

and returns the parsed values as JSON.
