# Lesson 142 — Cross-Origin Resource Sharing (CORS)

## 1. What is CORS?

**CORS** is a browser security mechanism that lets a server say **which other origins may read its responses**.

An **origin** is a combination of **scheme + hostname + port**.

Examples:

```text
http://localhost:3000   (React / Next.js)
http://localhost:5000   (Express API)
```

These are **different origins** because their ports differ.

## 2. Why do we need CORS?

Browsers enforce the **same-origin policy**: JavaScript from one origin generally cannot read responses from another origin unless the server allows it.

Example:

```js
// Frontend at http://localhost:3000
const res = await fetch("http://localhost:5000/api/users");
const data = await res.json();
```

Without appropriate CORS headers from the API, the browser blocks JavaScript from accessing the response.

**Important:** CORS is enforced by browsers, **not** by MongoDB or Node.js. Postman, curl, and backend-to-backend requests are not protected by browser CORS rules. CORS is not authentication.

## 3. How CORS works

```text
React app (localhost:3000)
  |
  | Request with Origin: http://localhost:3000
  v
Express API (localhost:5000)
  |
  | Access-Control-Allow-Origin: http://localhost:3000
  v
Browser allows React code to read the response
```

If the server doesn't allow that origin, the browser does not expose the response to JavaScript.

## 4. Simple vs preflighted requests

Some cross-origin requests are **simple** and do not need preflight, for example certain GET requests with only safelisted headers.

Other requests require a **preflight**: the browser sends an `OPTIONS` request first to ask permission.

A JSON POST with `Content-Type: application/json` usually triggers preflight.

```text
Browser -> OPTIONS /api/users
  Origin: http://localhost:3000
  Access-Control-Request-Method: POST
  Access-Control-Request-Headers: content-type

Server -> 204 + CORS permission headers

Browser -> POST /api/users
Server -> JSON response + CORS headers
```

The browser may cache a successful preflight. The real request is sent only if preflight checks succeed.

## 5. Express 5 setup with cors package

```bash
npm install cors
```

```js
import express from "express";
import cors from "cors";

const app = express();

app.use(cors({
  origin: "http://localhost:3000",
  methods: ["GET", "POST", "PATCH", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization"],
}));

app.use(express.json());

app.get("/api/users", (req, res) => {
  res.json({ users: [] });
});
```

The `cors` middleware handles normal CORS responses and preflight for app-wide use. Place it before your routes.

## 6. Cookies and credentials

If your frontend uses cookie-based authentication across origins:

```js
app.use(cors({
  origin: "https://app.example.com",
  credentials: true,
}));
```

Frontend:

```js
fetch("https://api.example.com/me", {
  credentials: "include",
});
```

**Rules:**
- Credentialed CORS requires a **specific allowed origin**, not `Access-Control-Allow-Origin: *`.
- The response also needs `Access-Control-Allow-Credentials: true`.
- Cookie sending still depends on cookie attributes (`SameSite`, `Secure`), browser privacy rules, and whether the sites are same-site/cross-site.
- `credentials: "include"` does not create a login session by itself.
- CSRF protection is still necessary where applicable.

## 7. Multiple frontend origins (allowlist)

```js
const allowedOrigins = new Set([
  "https://app.example.com",
  "https://admin.example.com",
]);

app.use(cors({
  origin(origin, callback) {
    if (!origin) return callback(null, false); // No browser Origin: no CORS headers
    callback(null, allowedOrigins.has(origin));
  },
  credentials: true,
}));
```

Never blindly reflect every `Origin` header when using credentials. Restrict trusted frontend origins.

## 8. Common CORS headers

| Header | Meaning |
| --- | --- |
| `Access-Control-Allow-Origin` | Which origin may read response |
| `Access-Control-Allow-Methods` | Allowed methods in preflight |
| `Access-Control-Allow-Headers` | Allowed request headers in preflight |
| `Access-Control-Allow-Credentials` | Whether credentialed cross-origin response is exposed |
| `Access-Control-Expose-Headers` | Extra response headers JS may read |
| `Access-Control-Max-Age` | How long preflight may be cached |
| `Vary: Origin` | Helps caches keep responses for different origins separate |

## 9. CORS errors: debugging checklist

1. Check exact frontend origin: protocol, hostname, and port.
2. Inspect browser DevTools Network for `OPTIONS` preflight.
3. Verify allowed methods and headers.
4. If cookies are used, check `credentials: "include"`, server `credentials: true`, and cookie policy.
5. Ensure reverse proxy or hosting layer isn't dropping CORS headers.
6. Check the actual backend error too: not every browser CORS-looking message means the root problem is CORS.

## 10. CORS is not access control

**Bad assumption:** "I allow only my React domain, so attackers cannot call my API."

Attackers can still call a public API using curl, Postman, or their own server. CORS only restricts **browser-based cross-origin reads**. Use authentication, authorization, validation, and rate limiting to secure the API.

## 11. Practice

1. Run React on port 3000 and Express on port 5000; inspect CORS headers.
2. Send a JSON POST and observe the OPTIONS preflight.
3. Configure an allowed origin and test an unlisted origin.
4. Explain why Postman can call an API even when a browser reports a CORS error.

## Interview-ready answer

CORS is a browser-enforced mechanism for controlled cross-origin response access. The API sends Access-Control headers; some requests require an OPTIONS preflight. It is not authentication or CSRF protection. For cookie-based requests, use exact origin allowlists, credentials settings, and correct cookie/CSRF protections.
