# Lesson 141 — Cross-Site Request Forgery (CSRF)

## 1. What is CSRF?

**CSRF** happens when a malicious website tricks a user's browser into sending an unwanted request to another website **where that user is already authenticated**.

Imagine you are logged in to a shopping app. You visit an unrelated malicious website. That website tries to make your browser send a request that changes your account details.

**Main problem:** Browsers may automatically attach cookies to requests. The server must not assume that a request is intentional just because it contains a valid session cookie.

## 2. Step-by-step flow

```text
User logs in to shop.example
    |
    v
Browser stores session cookie
    |
    v
User visits another website
    |
    v
That website tries to trigger a state-changing request
    |
    v
Browser may attach shop.example cookies (depending on SameSite rules)
    |
    v
If shop.example has no CSRF protection, the action may succeed
```

## 3. CSRF vs XSS

| CSRF | XSS |
| --- | --- |
| Tries to cause unwanted requests from another site | Runs unwanted code inside a vulnerable page |
| Often depends on automatically sent credentials | Depends on unsafe browser rendering |
| Defense: CSRF tokens, SameSite, Origin checks | Defense: output encoding, safe DOM APIs, CSP |

**Important:** XSS can defeat many CSRF defenses because script running on your own origin may access tokens or issue same-origin requests. Fix XSS too.

## 4. When is CSRF a risk?

Usually relevant when the browser **automatically sends authentication**, such as session cookies or cookies containing JWTs.

If a frontend explicitly attaches an access token in an `Authorization` header, an unrelated website generally cannot cause the browser to add that header automatically. That reduces the classic CSRF risk, but does not eliminate XSS, token theft, or other authentication issues.

## 5. SameSite cookies

```js
res.cookie("session", sessionId, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/",
});
```

- `HttpOnly`: scripts cannot read the cookie.
- `Secure`: cookie is sent over HTTPS.
- `SameSite: "Strict"`: strongest cross-site sending restriction, but can affect navigation/login flows.
- `SameSite: "Lax"`: useful default, but **not** a complete CSRF solution.
- `SameSite: "None"`: permits cross-site cookie sending and requires `Secure` in modern browsers; use only when necessary.

`SameSite` is based on **site**, not exactly the same concept as **origin**. Subdomains can be same-site while cross-origin.

## 6. Synchronizer CSRF token pattern

For a **server-side session**:

1. Server generates a strong random CSRF token and stores it in the user's session.
2. The legitimate frontend receives the token through a trusted page/API response.
3. Frontend includes the token in a custom header on state-changing requests.
4. Server compares the submitted token to the one stored in the session.
5. Missing or mismatched token → reject request.

### Express 5 example (illustrative session setup)

Assume `express-session` is already configured securely and `req.session` is available. Install `express-session` and use a production session store; do not use its default MemoryStore in production.

```js
import { randomBytes, timingSafeEqual } from "node:crypto";

app.get("/csrf-token", requireAuth, (req, res) => {
  if (!req.session.csrfToken) {
    req.session.csrfToken = randomBytes(32).toString("hex");
  }
  res.json({ csrfToken: req.session.csrfToken });
});

function requireCsrf(req, res, next) {
  const expected = req.session?.csrfToken;
  const provided = req.get("x-csrf-token");
  if (!expected || !provided) return res.sendStatus(403);

  const a = Buffer.from(expected, "utf8");
  const b = Buffer.from(provided, "utf8");
  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    return res.sendStatus(403);
  }
  next();
}

app.post("/account/email", requireAuth, requireCsrf, async (req, res) => {
  // Validate email, then update the authenticated user's account.
  res.json({ success: true });
});
```

**Frontend request:**

```js
const { csrfToken } = await fetch("/csrf-token", {
  credentials: "include",
}).then((r) => r.json());

await fetch("/account/email", {
  method: "POST",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "x-csrf-token": csrfToken,
  },
  body: JSON.stringify({ email: "new@example.com" }),
});
```

This illustrates the pattern, **not** a complete deployable session/authentication system. Configure trusted origins, session rotation, secure storage, login/logout handling, and consistent CSRF enforcement for every state-changing endpoint.

## 7. Double-submit cookie pattern

Another approach sends a CSRF token in a cookie and requires the client to echo it in a header/form field. The server compares them.

**Use a signed token bound to the authenticated session**, not a naive unsigned token comparison. Otherwise cookie injection and session confusion can weaken the protection. Follow a maintained, well-reviewed implementation.

## 8. Origin and Referer checks

For sensitive state-changing requests, the server can verify the `Origin` header against a strict allowlist (and use a carefully designed fallback policy where needed). `Sec-Fetch-Site` can add another signal.

**Never accept any origin just because it ends with your domain name** without proper hostname validation. Some clients omit Origin; decide explicitly how to handle that rather than silently trusting everything.

## 9. CORS is not CSRF protection

CORS mainly controls whether browsers let JavaScript **read** cross-origin responses and whether certain requests pass preflight. A malicious page may still trigger some cross-site requests, even if it cannot read the response.

A secure CORS configuration is useful, but **does not replace CSRF tokens and cookie policy**.

## 10. Safe HTTP methods and common mistakes

- `GET`, `HEAD`, `OPTIONS` should not change important state.
- Apply CSRF protection to state-changing operations: POST, PUT, PATCH, DELETE (and other mutation endpoints).
- Do not place CSRF tokens in URLs (logs/history can leak them).
- Do not assume JSON APIs are immune; consider request content types and browser behavior.
- Do not skip CSRF checks on password, email, or payment changes.

## 11. Practice

1. Explain why a session cookie may be attached without the user clicking your app.
2. Add CSRF token validation to a profile-update route.
3. Test missing and incorrect `x-csrf-token` values; expect 403.
4. Explain why `SameSite=Lax` and CORS alone do not guarantee CSRF protection.

## Interview-ready answer

CSRF tricks an authenticated browser into performing an unintended action. It is most relevant when browsers automatically send credentials such as cookies. Prevent it using CSRF tokens or a sound equivalent design, strict Origin checks, appropriate SameSite cookies, and safe HTTP method semantics.

## 12. Cross-origin frontend and backend: does CORS restrict where cookies are sent?

**Short answer: No.** An allowlist in Express CORS controls which origins may read credentialed responses (and which preflighted requests proceed). It is **not** a cookie-sending allowlist.

### Example deployment

| App | URL |
| --- | --- |
| React frontend | `https://frontend.com` |
| Express API | `https://api.backend.com` |
| Attacker page | `https://attacker.com` |

Backend:

```js
import cors from "cors";

app.use(cors({
  origin: ["https://frontend.com"],
  credentials: true,
}));

res.cookie("accessToken", token, {
  httpOnly: true,
  secure: true,
  sameSite: "none",
});
```

Frontend:

```js
fetch("https://api.backend.com/profile", {
  credentials: "include",
});
```

Because these domains are **cross-site**, `SameSite=None; Secure` allows the cookie to be eligible for cross-site requests. The browser may still block third-party cookies under its privacy settings or policies. The cookie also must match the **destination host/domain and path**; it is not sent to every website on the internet.

### What if attacker.com calls the API?

```js
fetch("https://api.backend.com/profile", {
  credentials: "include",
});
```

Even though `attacker.com` is **not** on the CORS allowlist, an eligible cookie **may be sent to api.backend.com**. CORS prevents the attacker's JavaScript from reading the response. For a request that requires a CORS preflight, the browser may block the actual request before it is sent; but simple requests can reach the server without preflight.

### HTML form example: CORS does not stop submission

```html
<form action="https://api.backend.com/change-email" method="POST">
  <input name="email" value="attacker@example.com" />
  <button type="submit">Submit</button>
</form>
```

If the server accepts form-encoded input, the browser sends an eligible authentication cookie, and the endpoint has no CSRF defense, the action may happen even though the attacker cannot read the response. **CORS is not an authorization or CSRF check.**

### Protect the backend: verify the request origin

```js
const allowedOrigins = new Set(["https://frontend.com"]);

function verifyOrigin(req, res, next) {
  const origin = req.get("Origin");
  if (!origin || !allowedOrigins.has(origin)) {
    return res.status(403).json({ message: "Forbidden origin" });
  }
  next();
}

app.post("/change-email", verifyOrigin, requireAuth, updateEmail);
```

This is a simplified, **fail-closed** example. Apply a consistent policy to all state-changing routes, account for trusted proxies and browser/header behavior, and use session-bound CSRF tokens where appropriate. The origin allowlist here is a **server-side rejection rule**, not merely a CORS response header.

### Cross-origin is not the same as cross-site

| Frontend | Backend | Relationship |
| --- | --- | --- |
| `https://app.example.com` | `https://api.example.com` | Cross-origin, **same-site** |
| `https://frontend.com` | `https://backend.com` | Cross-origin, **cross-site** |
| `http://localhost:3000` | `http://localhost:5000` | Cross-origin, **same-site** |

For same-site subdomains, `SameSite=Lax` or `Strict` may work even though CORS and `credentials: "include"` are still required for cross-origin `fetch`. Do not switch to `SameSite=None` simply because ports or subdomains differ.

### Remember the responsibilities

- **CORS:** controls cross-origin response access and preflight permissions.
- **SameSite:** determines cookie eligibility in cross-site contexts.
- **HttpOnly:** prevents JavaScript from reading the cookie, but not from causing requests that include it.
- **CSRF tokens / server-side Origin checks:** defend state-changing endpoints from unwanted cross-site requests.

**Interview takeaway:** `cors({ origin: ["https://frontend.com"], credentials: true })` does **not** mean the browser only sends cookies for requests initiated by that origin. Cookie policy and destination determine whether a cookie is sent; the backend must independently enforce CSRF protections.
