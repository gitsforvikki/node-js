# Lesson 126 — Cookies

## Why cookies matter

Cookies are often misunderstood as an authentication system.

They are not.

A cookie is a browser mechanism for storing small pieces of data and automatically attaching them to matching HTTP requests.

Cookies are commonly used to transport:

- session IDs
- access tokens
- refresh tokens
- CSRF tokens
- preferences

Understanding cookies deeply is essential for browser authentication.

---

## 1. Server sets cookie

Response:

```http
Set-Cookie: sessionId=abc123
```

Browser stores it.

---

## 2. Browser sends cookie

Future matching request:

```http
Cookie: sessionId=abc123
```

Browser decides whether to send the cookie based on cookie attributes.

---

## 3. Cookie mental model

```text
Server
  |
  | Set-Cookie
  v
Browser Cookie Store
  |
  | matching request
  v
Cookie header
  |
  v
Server
```

---

## 4. Cookie is not session

```text
Cookie
  -> client-side transport/storage

Session
  -> server-side auth state
```

A session commonly works as:

```text
cookie stores session ID
server stores session data
```

---

## 5. Cookie is not JWT

JWT can be stored in:

- cookie
- memory
- Authorization header flow

A cookie can store:

- JWT
- opaque session ID
- refresh token

These concepts are independent.

---

## 6. Set cookie in Express

```js
res.cookie(
  "refreshToken",
  token,
  {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
  }
);
```

---

## 7. Reading cookies

Express itself may require cookie parsing middleware/library if you want convenient access to parsed cookies.

Conceptually:

```js
req.cookies.refreshToken
```

Raw cookie data originates from:

```text
Cookie request header
```

---

## 8. Cookie size

Cookies are small.

Browsers impose limits per cookie/domain.

Do not store large objects in cookies.

Bad:

```json
{
  "fullUserProfile": {... huge ...}
}
```

---

## 9. Cookie sent on every matching request

If a cookie matches path/domain rules, browser may attach it automatically.

Large cookies increase request size repeatedly.

Keep auth cookies minimal.

---

## 10. Domain attribute

Cookie domain controls which hosts can receive it.

Host-only cookie:

```text
no Domain attribute
```

typically limits cookie to the host that set it.

Domain cookie:

```text
Domain=example.com
```

may be available to matching subdomains.

Broader scope increases exposure.

---

## 11. Prefer narrow domain scope

If only:

```text
api.example.com
```

needs auth cookie, avoid unnecessarily sharing it with all subdomains.

Least privilege applies to cookies too.

---

## 12. Path attribute

Example:

```http
Path=/auth
```

Cookie may only be sent to matching paths.

For refresh token, a narrow path such as:

```text
/auth/refresh
```

can reduce exposure.

But your logout flow may also need access depending on design.

---

## 13. Expires

Example:

```http
Expires=Wed, 07 Oct 2026 10:00:00 GMT
```

Sets an absolute expiration time.

---

## 14. Max-Age

Example:

```http
Max-Age=604800
```

Lifetime in seconds.

Often easier to manage than absolute dates.

---

## 15. Session cookie

If neither `Expires` nor `Max-Age` is set, cookie is generally treated as a session cookie.

Browser lifecycle behavior can vary, especially with session restoration.

Do not rely on this for strong security semantics without understanding browser behavior.

---

## 16. Deleting cookie

Set cookie with expiration in the past or Max-Age=0.

In Express:

```js
res.clearCookie(
  "refreshToken",
  {
    path:
      "/auth/refresh",
  }
);
```

Important:

Deletion should use matching cookie attributes such as path/domain.

---

## 17. Same name, different paths

Browsers can store multiple cookies with same name under different paths/domains.

This can create confusing bugs.

Use clear naming and scope.

---

## 18. Cookie encoding

Cookie values should be safe for HTTP header transport.

Libraries often handle encoding.

Do not put arbitrary unescaped data directly into cookie headers.

---

## 19. Signed cookies

Some frameworks/libraries support signed cookies.

Signed cookie means:

```text
value + signature
```

This detects tampering.

It does not encrypt the value.

Do not confuse:

```text
signed
```

with:

```text
encrypted
```

---

## 20. Cookie-based session example

```text
Login
  |
  v
server creates session row
  |
  v
Set-Cookie: sessionId=...
  |
  v
browser sends sessionId automatically
  |
  v
server loads session
```

---

## 21. JWT cookie example

```text
Login
  |
  v
server signs JWT
  |
  v
Set-Cookie: accessToken=JWT
  |
  v
browser sends JWT cookie
  |
  v
server verifies JWT
```

---

## 22. Refresh cookie example

Very common architecture:

```text
Access token
  -> Authorization header / memory

Refresh token
  -> HttpOnly Secure cookie
```

This keeps the long-lived token away from JavaScript.

---

## 23. Cookies and CORS

For cross-origin frontend/backend, client may need:

```js
fetch(url, {
  credentials:
    "include",
});
```

Server must allow credentials correctly.

---

## 24. Credentials and wildcard origin

You cannot combine credentialed CORS with:

```text
Access-Control-Allow-Origin: *
```

You must return a specific allowed origin.

---

## 25. Cookies and SameSite

SameSite controls cross-site sending behavior.

Modes:

- Strict
- Lax
- None

Covered deeply in Lesson 127.

---

## 26. Cookies and CSRF

Because cookies are automatically attached by browser, malicious sites can sometimes trigger authenticated requests.

This is why cookie-based auth needs CSRF thinking.

---

## 27. Cookies and XSS

HttpOnly cookies are not readable through JavaScript.

This reduces token theft.

But XSS can still make authenticated requests from the compromised page.

HttpOnly does not "solve XSS."

---

## 28. Cookie security principles

Use:

- narrow domain
- narrow path where practical
- HttpOnly for secrets
- Secure in production
- appropriate SameSite
- short expiration
- rotation for long-lived auth secrets

---

## 29. Do not store sensitive plaintext data

Avoid storing:

- passwords
- card details
- private personal data
- large profile objects

Cookies are sent repeatedly and may be exposed through browser/device compromise.

---

## 30. Cookie prefixes

Modern browsers support security-oriented cookie name prefixes such as:

```text
__Secure-
__Host-
```

`__Host-` generally enforces strong scoping requirements such as Secure and no Domain attribute.

Useful for hardening sensitive cookies.

---

## 31. Cookie scope example

Safer refresh cookie:

```text
Name: __Host-refreshToken
Secure
HttpOnly
Path=/
No Domain
SameSite=Lax/Strict/None depending architecture
```

Actual SameSite choice depends on frontend/backend site relationship.

---

## 32. Reverse proxy consideration

If HTTPS terminates at proxy and Node sees HTTP internally, cookie configuration may require correct trust-proxy setup.

Otherwise secure-cookie logic can behave unexpectedly.

---

## 33. Common mistakes

### Mistake 1
Thinking cookie = session.

### Mistake 2
Broad Domain unnecessarily.

### Mistake 3
Forgetting credentials on cross-origin fetch.

### Mistake 4
Wildcard CORS with credentials.

### Mistake 5
Deleting cookie with mismatched path/domain.

### Mistake 6
Storing huge/sensitive objects.

---

## 34. Interview questions

### What is a cookie?

A browser-managed key/value value that can be automatically sent with matching HTTP requests.

### Cookie vs session?

Cookie is client-side transport/storage; session is server-side state.

### Can JWT be stored in cookie?

Yes.

### Why are cookies involved in CSRF?

Because browsers may send them automatically on cross-site requests.

### Does signed cookie mean encrypted?

No.

---

## 35. Strong interview answer

> A cookie is a browser-managed HTTP mechanism, not an authentication strategy by itself. The server sends Set-Cookie, the browser stores the value, and later attaches it to matching requests based on domain, path, expiration, SameSite, and Secure rules. Cookies are useful for session IDs and refresh tokens, but because they are automatically sent, cookie-based authentication must account for CSRF and should use tight scope and security attributes.

---

## Interview-Ready Summary

```text
Set-Cookie
   |
   v
Browser stores
   |
   v
Cookie header

Cookie controls:
Domain
Path
Expires
Max-Age
HttpOnly
Secure
SameSite

Cookie != session
Cookie != JWT
```

## Practical Task

Design cookies for:

1. session ID
2. access token
3. refresh token
4. CSRF token

Explain:
- domain
- path
- expiry
- security flags
