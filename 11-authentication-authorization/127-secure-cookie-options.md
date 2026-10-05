# Lesson 127 — HTTP-only, Secure and SameSite Cookies

## Why this lesson is extremely important

These three cookie attributes are among the most important browser-authentication security controls:

```text
HttpOnly
Secure
SameSite
```

Interviewers often ask them together because they protect against different threats.

A strong answer should clearly explain:

- what each attribute does
- what each does NOT do
- CSRF vs XSS
- same-site vs same-origin
- cross-domain frontend/backend behavior
- production cookie configuration

---

# Part 1 — HttpOnly

## 1. What does HttpOnly do?

Example:

```http
Set-Cookie: refreshToken=abc; HttpOnly
```

JavaScript cannot normally read the cookie through:

```js
document.cookie
```

This protects secret cookies from direct JavaScript access.

---

## 2. Why HttpOnly matters

Suppose site has XSS.

Without HttpOnly:

```js
const token =
  document.cookie;
```

Attacker may steal long-lived auth cookies.

With HttpOnly:

```text
JavaScript cannot directly read protected cookie
```

---

## 3. What HttpOnly does NOT do

HttpOnly does not stop malicious JavaScript from making requests as the user.

If attacker controls page script:

```js
fetch(
  "/api/transfer",
  {
    method: "POST",
  }
);
```

Browser can still attach HttpOnly cookies.

So:

```text
HttpOnly reduces credential theft
but does not eliminate XSS impact
```

---

# Part 2 — Secure

## 4. What does Secure do?

```http
Set-Cookie: refreshToken=abc; Secure
```

Browser sends cookie only over secure HTTPS connections.

This prevents cookie transmission over plain HTTP.

---

## 5. Why Secure matters

Without Secure:

```text
HTTP connection
   |
   v
cookie may travel unencrypted
```

Network attacker could capture it.

---

## 6. Production rule

Authentication cookies should generally use:

```text
Secure=true
```

in production.

---

## 7. Local development

Browsers have special localhost behavior in some contexts, but production configuration should always assume HTTPS.

Do not weaken production cookies because local development is inconvenient.

---

# Part 3 — SameSite

## 8. What does SameSite do?

SameSite controls when cookies are sent with **cross-site** requests.

Values:

```text
Strict
Lax
None
```

---

## 9. Same-site vs same-origin

Critical interview concept.

Origin is:

```text
scheme + host + port
```

Example:

```text
https://app.example.com
https://api.example.com
```

These are different origins.

But they may still be same-site because they share the registrable domain:

```text
example.com
```

CORS is origin-based.

SameSite is site-based.

This distinction causes many bugs.

---

## 10. SameSite=Strict

Strongest cross-site restriction.

Cookie generally stays within same-site navigation/request context.

Good for highly sensitive cookies when cross-site flows are unnecessary.

Trade-off:

External link/login/payment flows may become inconvenient.

---

## 11. SameSite=Lax

A balanced default in many applications.

It blocks many cross-site subrequests while still allowing some top-level safe navigations.

Often practical for standard web sessions.

---

## 12. SameSite=None

Allows cross-site cookie sending.

Must generally be paired with:

```text
Secure
```

Example:

```http
SameSite=None; Secure
```

Used when frontend/backend truly require cross-site credential behavior.

---

## 13. Why SameSite=None is riskier

Because browser can attach cookie in cross-site contexts.

That increases CSRF exposure.

If you need SameSite=None, add stronger CSRF defenses.

---

# Part 4 — CSRF

## 14. What is CSRF?

CSRF:

```text
Cross-Site Request Forgery
```

Attack:

```text
User logged into bank.com
      |
      v
visits attacker.com
      |
      v
attacker causes request to bank.com
      |
      v
browser attaches bank cookie automatically
```

If server lacks CSRF protection, action may succeed.

---

## 15. Why cookies create CSRF risk

Because browser automatically sends matching cookies.

Attacker does not need to know cookie value.

---

## 16. SameSite as CSRF defense

```text
Strict
  -> strongest protection

Lax
  -> blocks many cross-site state-changing contexts

None
  -> no SameSite protection
```

SameSite is helpful, but do not treat it as the only defense in every architecture.

---

## 17. CSRF token pattern

Server issues random token.

Client sends token in custom header:

```http
X-CSRF-Token: ...
```

Attacker site generally cannot read token due to browser same-origin protections.

Server verifies:

```text
cookie/session
+
CSRF token
```

---

## 18. Double-submit cookie pattern

Possible design:

```text
CSRF cookie
+
X-CSRF-Token header
```

Server checks both match.

Use strong random tokens and understand signing requirements.

---

## 19. Origin checking

For sensitive state-changing endpoints, server can validate:

```text
Origin
```

and sometimes:

```text
Referer
```

against trusted frontend origins.

Useful defense in depth.

---

# Part 5 — XSS vs CSRF

## 20. XSS

XSS:

```text
attacker JavaScript executes inside your site
```

Potential impact:
- steal localStorage token
- call APIs
- read page data
- change UI

---

## 21. CSRF

CSRF:

```text
attacker site causes victim browser to send authenticated request
```

Attacker script is not necessarily running inside your site.

---

## 22. Cookie flag mapping

```text
HttpOnly
  -> reduces JS access to cookie
  -> helps against token theft via XSS

Secure
  -> HTTPS-only transmission
  -> helps against network interception

SameSite
  -> controls cross-site sending
  -> helps against CSRF
```

This mapping is interview gold.

---

# Part 6 — Practical Configurations

## 23. Same-site frontend/backend

Example:

```text
https://app.example.com
https://api.example.com
```

Different origins, often same-site.

Typical cookie may use:

```text
HttpOnly
Secure
SameSite=Lax
```

depending on exact request/navigation behavior.

CORS still needs configuration because origins differ.

---

## 24. Truly cross-site frontend/backend

Example:

```text
https://myapp.vercel.app
https://api.examplebackend.com
```

These are cross-site.

Cookie may require:

```text
SameSite=None
Secure
```

Then add:

- credentials include
- strict CORS
- CSRF defense
- origin checks

---

## 25. Express cookie example

```js
res.cookie(
  "refreshToken",
  refreshToken,
  {
    httpOnly: true,
    secure:
      process.env.NODE_ENV ===
      "production",
    sameSite:
      "none",
    maxAge:
      7 * 24 * 60 * 60 * 1000,
    path:
      "/auth/refresh",
  }
);
```

This is only an example.

Your actual SameSite setting depends on site topology.

---

## 26. Cross-origin fetch

Client:

```js
fetch(
  "https://api.example.com/auth/refresh",
  {
    method: "POST",
    credentials:
      "include",
  }
);
```

Axios:

```js
axios.post(
  url,
  body,
  {
    withCredentials: true,
  }
);
```

---

## 27. Server CORS

Credentialed CORS:

```js
app.use(
  cors({
    origin:
      "https://app.example.com",
    credentials: true,
  })
);
```

Do not use:

```text
origin: *
credentials: true
```

That is invalid/insecure design.

---

## 28. SameSite=None requires Secure

Modern browsers generally reject or restrict:

```text
SameSite=None
without Secure
```

Remember this for interviews and deployments.

---

## 29. Domain pitfalls

If you set:

```text
Domain=.example.com
```

cookie may be exposed to more subdomains than needed.

Prefer host-only cookies where possible.

---

## 30. Path pitfalls

If refresh cookie uses:

```text
Path=/auth/refresh
```

then a logout endpoint at:

```text
/auth/logout
```

may not receive it.

Design cookie path and logout behavior deliberately.

---

## 31. Clearing cookie

To reliably clear:

```js
res.clearCookie(
  "refreshToken",
  {
    httpOnly: true,
    secure: true,
    sameSite: "none",
    path:
      "/auth/refresh",
  }
);
```

Path/domain must match relevant cookie scope.

---

## 32. __Host- prefix

Example:

```text
__Host-refreshToken
```

Browsers enforce stronger requirements such as:

- Secure
- no Domain attribute
- Path=/

This can harden auth cookies.

---

## 33. __Secure- prefix

Example:

```text
__Secure-refreshToken
```

Requires Secure context.

Less restrictive than `__Host-`.

---

## 34. CORS is not CSRF protection by itself

Critical point:

CORS controls whether JavaScript can **read** cross-origin responses.

It is not a universal blocker of sending requests.

Do not say:

> CORS prevents CSRF.

That is incorrect.

---

## 35. Preflight is not a security boundary

Browsers may preflight some requests.

But your security model should rely on:

- authentication
- CSRF protection
- authorization
- SameSite
- origin validation

not on "the browser will preflight it."

---

## 36. Cookie expiration and token expiration

If cookie lasts:

```text
7 days
```

but refresh token expires:

```text
1 day
```

browser may keep sending an already-invalid token.

Align lifetimes intentionally.

---

## 37. Secure architecture example

```text
Login
  |
  v
Access token -> short-lived
Refresh token -> HttpOnly Secure cookie
  |
  v
API request
  |
  +--> Bearer access token
  |
  v
Access expires
  |
  v
POST /auth/refresh
  |
  +--> browser sends refresh cookie
  +--> CSRF/origin checks
  |
  v
rotate refresh token
  |
  v
issue new access token
```

---

## 38. Common mistakes

### Mistake 1
Thinking HttpOnly prevents CSRF.

### Mistake 2
Thinking SameSite prevents XSS.

### Mistake 3
SameSite=None without Secure.

### Mistake 4
Wildcard CORS with credentials.

### Mistake 5
Wrong Domain/Path.

### Mistake 6
Using localStorage for long-lived refresh token.

### Mistake 7
Assuming CORS alone prevents CSRF.

---

## 39. Interview questions

### What does HttpOnly do?

Prevents JavaScript from directly reading the cookie.

### What does Secure do?

Restricts cookie transmission to HTTPS.

### What does SameSite do?

Controls whether cookies are sent in cross-site contexts.

### Strict vs Lax vs None?

Strict is strongest cross-site restriction, Lax is balanced, None allows cross-site sending and requires Secure.

### Same-site vs same-origin?

Origin includes scheme, host, and port; site is based on scheme plus registrable domain semantics.

### Does HttpOnly stop XSS?

No. It prevents direct cookie reading, but XSS can still perform authenticated actions.

### Does CORS stop CSRF?

No.

---

## 40. Strong interview answer

> HttpOnly, Secure, and SameSite protect different aspects of cookie security. HttpOnly prevents JavaScript from directly reading the cookie, Secure ensures it is transmitted only over HTTPS, and SameSite controls cross-site sending to reduce CSRF risk. For a cross-site browser architecture, I may need SameSite=None with Secure, credentialed CORS, and explicit CSRF/origin protection. I also keep cookie domain and path as narrow as possible and never treat CORS alone as CSRF protection.

---

## Interview-Ready Summary

```text
HttpOnly
  -> JS cannot read
  -> token theft protection

Secure
  -> HTTPS only
  -> transport protection

SameSite
  -> cross-site sending policy
  -> CSRF protection

Strict
Lax
None + Secure

Remember:
XSS != CSRF
CORS != CSRF protection
```

## Section 11 Progress Map

```text
Authentication vs Authorization
        |
        v
Password Hashing
        |
        v
bcrypt
        |
        v
JWT
        |
        v
Access Tokens
        |
        v
Refresh Tokens
        |
        v
Cookies
        |
        v
HttpOnly / Secure / SameSite
        |
        v
Next:
Session Authentication
JWT vs Sessions
RBAC
Auth Middleware
OAuth
Password Reset
Email Verification
Token Rotation / Revocation
```

## Practical Task

Design two browser auth architectures:

### Architecture A
Frontend and backend under same site.

### Architecture B
Frontend and backend on completely different sites.

For each define:

- access token storage
- refresh token storage
- cookie options
- CORS
- CSRF protection
- logout flow
