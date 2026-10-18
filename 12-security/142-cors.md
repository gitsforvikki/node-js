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

A **CORS preflight** is a permission check the browser performs **before** sending certain cross-origin requests. The browser automatically sends an `OPTIONS` request to ask whether the intended method and headers are allowed. You do **not** write this OPTIONS call in React.

### 4.1 Our setup: two different origins

| Application | URL |
| --- | --- |
| React frontend | `http://localhost:3000` |
| Express API | `http://localhost:5000` |

The ports differ, so the origins differ.

### 4.2 Simple GET request — no preflight

```js
// Runs in React at localhost:3000
const response = await fetch("http://localhost:5000/api/users");
const users = await response.json();
```

For this ordinary GET request with no custom non-safelisted headers, the browser can send the **actual GET directly**:

```text
React browser (3000)                 Express API (5000)
       |                                     |
       |-------- GET /api/users ----------->|
       |  Origin: http://localhost:3000      |
       |                                     |
       |<--- 200 + JSON + CORS headers ------|
       |                                     |
       v
React can read the response if CORS permits it.
```

Example route:

```js
app.get("/api/users", (req, res) => {
  res.json([{ id: 1, name: "Rahul" }, { id: 2, name: "Vikash" }]);
});
```

**Important:** No preflight does **not** mean no CORS check. The browser still checks the response's CORS headers. Without permission, the GET may have already reached the server, but JavaScript cannot read the response.

### 4.3 JSON POST — preflight required

```js
fetch("http://localhost:5000/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Vikash" }),
});
```

Why preflight? `application/json` is **not** a CORS-safelisted Content-Type. The browser asks permission before sending the actual POST.

**Step 1 — Browser automatically sends OPTIONS:**

```http
OPTIONS /api/users HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
```

- `Origin`: Which website initiated the request?
- `Access-Control-Request-Method`: Which HTTP method will the actual request use?
- `Access-Control-Request-Headers`: Which non-safelisted headers will it include?

**Step 2 — Express replies with permission headers:**

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET,POST
Access-Control-Allow-Headers: Content-Type
```

`204` means successful response with no body. The exact successful status can vary; the important part is the correct CORS headers.

**Step 3 — Browser sends actual POST only if preflight succeeds:**

```http
POST /api/users HTTP/1.1
Origin: http://localhost:3000
Content-Type: application/json

{"name":"Vikash"}
```

**Step 4 — Server returns JSON and CORS headers:**

```http
HTTP/1.1 201 Created
Access-Control-Allow-Origin: http://localhost:3000
Content-Type: application/json

{"message":"User created","user":{"name":"Vikash"}}
```

The browser also checks CORS on this **actual response**.

### 4.4 Complete Express 5 example

```js
import express from "express";
import cors from "cors";

const app = express();

app.use(cors({
  origin: "http://localhost:3000",
  methods: ["GET", "POST"],
  allowedHeaders: ["Content-Type"],
}));

app.use(express.json());

app.get("/api/users", (req, res) => {
  res.json([{ id: 1, name: "Rahul" }]);
});

app.post("/api/users", (req, res) => {
  res.status(201).json({
    message: "User created",
    user: req.body,
  });
});

app.listen(5000);
```

With app-wide `cors()`, the middleware normally handles the preflight OPTIONS request automatically. Place it before routes. This example focuses on CORS; a production API still needs input validation and appropriate authentication/authorization.

### 4.5 Which requests typically need preflight?

| Cross-origin request | Preflight? |
| --- | --- |
| GET without custom headers | Usually no |
| POST with `application/x-www-form-urlencoded` | Usually no |
| POST with `text/plain` | Usually no |
| POST with `application/json` | Yes |
| PUT / PATCH / DELETE | Yes |
| GET with `Authorization` header | Yes |

These are common examples, assuming no other request features or headers change the classification. Simple requests use safelisted methods (GET, HEAD, POST) and meet additional header/content-type restrictions.

### 4.6 What if preflight is rejected?

Suppose the backend only allows GET:

```js
app.use(cors({
  origin: "http://localhost:3000",
  methods: ["GET"],
}));
```

A preflight asking to send POST will **not** receive POST permission. The browser blocks the **actual POST**. This is browser behavior: Postman, curl and backend-to-backend clients are not required to enforce browser CORS rules.

### 4.7 Preflight is not CSRF protection

A simple cross-site form POST using `application/x-www-form-urlencoded` can reach the server **without preflight**. If eligible authentication cookies accompany it and the backend has no CSRF defense, an unwanted state-changing action may happen. **CORS is not authorization or a replacement for CSRF tokens / server-side Origin checks.**

### 4.8 Why don't I see OPTIONS before every POST?

A successful preflight may be cached according to `Access-Control-Max-Age` and browser-specific limits. Therefore, the browser need not repeat OPTIONS for every matching request.

**Mental model:**

- **Simple request:** send actual request directly → check CORS on response.
- **Preflighted request:** send OPTIONS → check permissions → send actual request → check CORS on response.
- **Interview answer:** A CORS preflight is a browser-generated OPTIONS request that checks whether a server allows the intended cross-origin method and headers before the actual request is sent.

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
