# Lesson 124 — Access Tokens

## Why access tokens matter

Access tokens are one of the most important building blocks in modern authentication systems.

They answer a practical question:

> How does a client prove its identity on every protected API request without sending the user's password again?

A good access-token design should balance:

- security
- performance
- revocation
- usability
- scalability
- short-lived trust

This lesson focuses on the production and interview concepts you should understand deeply.

---

## 1. What is an access token?

**Access Token ek temporary credential hai** jo client protected APIs ko access karne ke liye server ko bhejta hai.

Imagine karo tum React + Express me e-commerce app bana rahe ho. User login karta hai aur phir `/profile`, `/orders`, `/dashboard` open karta hai.

**Problem:** Har API request me email aur password dobara bhejna secure/practical nahi hai. Server kaise jaane ki request authenticated user ki hai?

**Solution:** Login successful hone ke baad server access token issue kar sakta hai. Client future API calls ke saath token bhejta hai.

```http
GET /api/profile
Authorization: Bearer <ACCESS_TOKEN>
```

Server token verify karke user ko identify karta hai, **phir separately check karta hai ki requested resource ka permission hai ya nahi**.

## 2. Access token purpose — complete login flow

```text
1. User -> POST /api/login (email + password)
2. Express -> DB se user find + saved password hash verify
3. Valid login -> server creates short-lived access token
4. Client receives access token
5. Client -> GET /api/profile with Bearer access token
6. Server -> verifies token + checks authorization -> response
```

Example login request:

```http
POST /api/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "hello123"
}
```

**JWT is one possible access-token format**, not the definition of access token. An opaque random token can also be an access token. Password hashing and JWT signing are separate jobs.
---

## 3. Access token is not always JWT

Important interview point:

An access token can be:

- JWT
- opaque random token
- reference token

JWT is only one format.

---

## 4. JWT access token

A JWT access token may contain claims:

```json
{
  "sub": "usr_123",
  "role": "user",
  "iat": 1710000000,
  "exp": 1710000900
}
```

The API verifies:

- signature
- expiration
- issuer
- audience
- allowed algorithm

---

## 5. Opaque access token

Example:

```text
3b9f8f8d8c7...
```

The token itself contains no readable claims.

Server looks it up:

```text
token
  |
  v
session/token store
  |
  v
user identity + permissions
```

---

## 6. JWT vs opaque access token

JWT:

```text
self-contained claims
fast distributed verification
revocation harder
stale claims possible
```

Opaque token:

```text
server-side lookup
easy immediate revocation
requires shared token/session store
```

Neither is universally superior.

---

## 7. Why access tokens should be short-lived

**Bearer token** ka matlab: jiske paas valid token hai, woh use present kar sakta hai. Agar attacker token chura le, signature usse **stolen token reuse** karne se nahi rokta.

```text
Token stolen -> attacker sends same valid token -> possible account access
```

So use **short expiry**, HTTPS, secure storage and revocation controls where appropriate.

Example: 15-minute access token ka exposure window 30-day token se kaafi chhota hota hai. Lekin stolen token ko immediately revoke karne ke liye additional server-side mechanism lag sakta hai.

## 8. Typical lifetime

Common examples: **5 minutes, 15 minutes, 30 minutes, 1 hour**. Koi universal best number nahi hai.

Expiry choose karte time consider karo:
- App sensitivity (banking vs low-risk content)
- User experience
- Refresh-token/session design
- Risk tolerance

**15 minutes** is lesson me sirf example hai, compulsory rule nahi.
---

## 9. Access token request flow

```text
Client
  |
  | Authorization: Bearer token
  v
API
  |
  v
extract token
  |
  v
verify token
  |
  v
authenticate user
  |
  v
authorize request
  |
  v
controller
```

Authentication and authorization remain separate.

---

## 10. Bearer token meaning

Bearer means:

> Whoever possesses this token can use it.

That means access tokens must be protected like credentials.

If stolen, the attacker can act as the user until token expiry or revocation.

---

## 11. Access-token claims

Good claims:

```text
sub
iss
aud
exp
iat
jti
small role/scope claims
```

Avoid huge payloads.

---

## 12. Minimal claims principle

Bad:

```json
{
  "sub": "usr_123",
  "email": "...",
  "address": "...",
  "preferences": {...},
  "permissions": ["hundreds..."]
}
```

Problems:

- larger token
- stale information
- privacy exposure
- bandwidth cost

Keep claims minimal.

---

## 13. Role in access token

Possible:

```json
{
  "role": "admin"
}
```

Works well when:

- role changes infrequently
- access token TTL is short

For sensitive actions, you may still query current authorization state.

---

## 14. Scope

OAuth-style token may contain:

```text
scope:
read:profile
write:orders
```

Authorization can then check whether requested action is allowed.

---

## 15. Audience

```text
aud
```

should identify intended recipient.

Example:

```text
careerloop-api
```

This prevents a token intended for service A from being blindly accepted by service B.

---

## 16. Issuer

```text
iss
```

identifies who issued the token.

Example:

```text
https://auth.example.com
```

Always validate expected issuer when applicable.

---

## 17. jti

```text
jti
```

is a unique token identifier.

Useful for:

- audit
- denylist
- tracing
- replay handling

---

## 18. Access token storage in browser — React app me kaha rakhein?

Browser me teen common approaches hain. Har option ke security trade-offs hain.

| Storage | Benefit | Main trade-off |
| --- | --- | --- |
| **JavaScript memory** (module/React state) | Page refresh par token persist nahi hota | Reload par token lost; active XSS still misuse kar sakta hai |
| **HttpOnly cookie** | JavaScript directly token read nahi kar sakta | Browser automatically cookie bhejta hai; CSRF protection important |
| **localStorage** | Easy and survives reload | XSS me attacker token read/steal kar sakta hai |

### Option A — localStorage (simple, but risky)

```js
localStorage.setItem("accessToken", accessToken);

const token = localStorage.getItem("accessToken");

await fetch("/api/profile", {
  headers: { Authorization: `Bearer ${token}` },
});
```

Convenient hai, lekin production me persistent JS-readable token storage **generally avoid** karna better hai, especially when XSS risk matters.

### Option B — HttpOnly cookie

Server cookie set karta hai:

```js
res.cookie("accessToken", accessToken, {
  httpOnly: true,
  secure: true,      // production HTTPS
  sameSite: "lax",  // depends on your deployment
  maxAge: 15 * 60 * 1000,
});
```

Client:

```js
await fetch("/api/profile", {
  credentials: "include",
});
```

Matching requests par browser cookie automatically send karta hai. React JavaScript **HttpOnly** cookie ko directly read nahi kar sakta. Backend cookie se token extract karke verify karega (Bearer-header extraction se alag). Cross-origin requests me correct CORS/credentials settings bhi chahiye.

**Caution:** HttpOnly XSS ke sab effects ko stop nahi karta; malicious script still authenticated requests trigger kar sakta hai. Cookies ke saath CSRF controls (SameSite, origin checks, anti-CSRF token as needed) important hain.

### Option C — In-memory access token + HttpOnly refresh cookie

React SPA ke liye useful pattern, especially jab API `Authorization: Bearer` expect karti ho.

```text
LOGIN
  |
  +--> short-lived access token -> React JS memory
  |
  +--> longer-lived refresh token -> HttpOnly Secure cookie

PROTECTED API
  React memory access token -> Authorization: Bearer -> Express

AFTER ACCESS TOKEN EXPIRES / PAGE RELOAD
  Browser sends refresh cookie -> POST /auth/refresh
  Server validates refresh session, rotates token as appropriate
  Server returns NEW access token -> React memory
```

**Why?** Access token persistent JavaScript storage me nahi rehta. Refresh token JavaScript ko directly accessible nahi hota. But refresh endpoint still needs CSRF protection, refresh-token rotation/revocation and careful CORS/origin rules.

Page refresh par memory clear ho jaati hai. Valid refresh cookie ho to application naya access token le sakti hai. Invalid/expired refresh session ho to user ko login karna padega.

**Alternative:** Same-site traditional web apps me secure HttpOnly cookie-backed server session simpler ho sakta hai. Har app me JWT compulsory nahi hai.

## 19. Access token in Authorization header

```http
GET /api/profile
Authorization: Bearer eyJ...
```

Browser automatically custom Bearer header attach nahi karta. Client JavaScript, mobile app, ya API client explicitly bhejta hai.

Common for SPAs, mobile apps, service-to-service requests.

## 20. Access token in cookie

```http
Cookie: accessToken=...
```

Browser matching cookie automatically bhej sakta hai. Server must read and verify it. **HttpOnly** JS reading ko block karta hai; **Secure** HTTPS-only transmission ke liye hai (production).

## 21. Cookie-based access token and CSRF

Cookie automatically sent hone ke karan attacker malicious website se authenticated requests trigger karne ki koshish kar sakta hai.

Defenses:
- Appropriate `SameSite` setting
- Server-side `Origin` checks
- Anti-CSRF token for state-changing requests when needed
- Restrictive CORS and correct cookie domain/path settings

`SameSite: "lax"` every architecture ke liye sufficient nahi hai.

## 22. Authorization header and CSRF

`Authorization: Bearer` automatically cross-site attach nahi hota, so traditional cookie-based CSRF risk usually lower hai. **But** JS-readable token storage se XSS-based token theft ka risk badhta hai.

**Conclusion:** One universally safe storage location nahi hai. Threat model aur app architecture ke according choose karo.
---

## 23. Practical React + Express example — issuing and using access tokens

Install: `npm install jsonwebtoken bcrypt`.

### Step 1: Express login controller (illustrative)

```js
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";

export async function loginController(req, res) {
  const { email, password } = req.body;
  const user = await userRepository.findByEmailWithPassword(email);

  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ message: "Invalid credentials" });
  }

  const accessToken = jwt.sign(
    { sub: String(user.id) },
    process.env.JWT_SECRET,
    { algorithm: "HS256", expiresIn: "15m" }
  );

  return res.status(200).json({ accessToken });
}
```

`userRepository` is a placeholder for your database layer. This demonstrates issuance only; production login also needs input validation, rate limiting, HTTPS and secure refresh/session handling.

### Step 2: React in-memory token

```js
// authToken.js (simplified in-memory storage)
let accessToken = null;

export function setAccessToken(token) {
  accessToken = token;
}

export async function getProfile() {
  if (!accessToken) throw new Error("Login required");

  return fetch("/api/profile", {
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
  });
}
```

After successful login, call `setAccessToken(response.accessToken)`. Module memory clears on page reload; for continuity, use a securely designed refresh flow.

### Step 3: Express 5 authentication middleware

```js
import jwt from "jsonwebtoken";

export function authenticate(req, res, next) {
  const header = req.headers.authorization;

  if (!header?.startsWith("Bearer ")) {
    return res.status(401).json({ message: "Authentication required" });
  }

  const token = header.slice(7);

  // jwt.verify throws on invalid/expired token.
  // Express 5 error middleware can handle this; map JWT errors to 401.
  const payload = jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ["HS256"],
    // issuer: "your-auth-service",
    // audience: "your-api",
  });

  req.auth = { userId: payload.sub };
  next();
}
```

**Important Express 5 detail:** Express 5 automatically forwards thrown errors from async route handlers/middleware returning promises, but **synchronous middleware errors are also caught by Express**. Centralized error middleware should map `TokenExpiredError` and `JsonWebTokenError` to HTTP 401, instead of accidentally returning 500. `req.auth` only establishes authenticated identity; controllers/authorization middleware must still enforce ownership/roles.

### Step 4: Protected route

```js
app.get("/api/profile", authenticate, async (req, res) => {
  const user = await userRepository.findPublicById(req.auth.userId);
  if (!user) return res.sendStatus(404);
  res.json(user);
});
```

This is a learning example, not a full production auth system.
---

## 24. Do not trust decoded token without verification

Wrong:

```text
decode(token)
   |
   v
trust userId
```

Correct:

```text
verify signature + claims
   |
   v
trust claims
```

---

## 25. Expired token — what happens after 15 minutes?

```text
00:00 -> access token issued
00:15 -> access token expires
After expiry -> protected API returns 401 Unauthorized
```

Client may call `POST /auth/refresh` using a valid refresh credential and obtain a **new access token**.

```text
API -> 401 expired token
       |
       v
POST /auth/refresh (refresh cookie)
       |
   +---+---+
   |       |
 valid   invalid / expired
   |       |
   v       v
new access token  login again
   |
   v
retry original request once
```

Implement refresh rotation/revocation and protect the refresh endpoint from CSRF. Avoid infinite refresh/retry loops. If no refresh credential exists, login again.

## 26. Invalid token

Examples: bad signature, malformed token, wrong issuer/audience, unsupported algorithm, or expired token.

Return **401 Unauthorized** for authentication failure. **403 Forbidden** is generally for an authenticated user who lacks permission.
---

## 27. Token revocation challenge

JWT remains valid until expiration unless server checks revocation state.

Possible strategies:

- short expiration
- token denylist
- token version
- session record
- refresh-session revocation

---

## 28. Token version pattern

User:

```json
{
  "tokenVersion": 8
}
```

Token:

```json
{
  "sub": "usr_1",
  "ver": 8
}
```

If password reset:

```text
tokenVersion -> 9
```

Old access tokens can be rejected after DB lookup.

Trade-off:

```text
verification no longer fully stateless
```

---

## 29. High-risk action strategy

For sensitive actions:

```text
change password
delete account
payment
admin operation
```

you may require:

- fresh authentication
- recent login
- DB role check
- MFA

A valid token alone may not be enough.

---

## 30. Access token theft

Attacker can steal through:

- XSS
- malware
- browser extension
- logs
- analytics
- insecure network
- compromised device

Defenses:

- HTTPS
- short TTL
- HttpOnly where appropriate
- never log tokens
- CSP/XSS protection
- refresh rotation

---

## 31. Never log tokens

Bad:

```js
console.log(
  req.headers.authorization
);
```

Logs are often widely accessible.

Redact tokens.

---

## 32. Token size

JWT is sent repeatedly on requests.

Large JWT means:

```text
larger HTTP headers
more bandwidth
possible proxy/header limits
```

Keep tokens small.

---

## 33. Logout

If access token is short-lived:

```text
logout
   |
   +--> clear client token
   +--> revoke refresh session
   |
   v
access token dies naturally soon
```

For immediate revocation, server-side state is required.

---

## 34. Access token vs refresh token

| Feature | Access token | Refresh token |
| --- | --- | --- |
| Purpose | Call protected APIs | Obtain new access token |
| Lifetime | Usually short | Usually longer |
| Where sent? | Protected endpoints | Refresh endpoint only |
| Browser storage | Memory or secure HttpOnly cookie | Prefer secure HttpOnly cookie in this SPA design |
| Must be JWT? | No | No |
| Revocation | Depends on design | Should support revocation and rotation |

**Mental model:** Access token = short-term API pass. Refresh token = protected credential for obtaining another pass. Refresh-token design is covered in **[Lesson 125 — Refresh Tokens](125-refresh-tokens.md)**.
---

## 35. Service-to-service access tokens

Machine clients may use:

- client credentials
- signed JWT
- mTLS
- API tokens

Do not reuse end-user token architecture blindly for service identities.

---

## 36. Common mistakes

### Mistake 1
Very long access-token lifetime.

### Mistake 2
Sensitive data in payload.

### Mistake 3
Decode without verify.

### Mistake 4
No audience/issuer validation.

### Mistake 5
Logging bearer tokens.

### Mistake 6
Using access token as refresh token.

### Mistake 7
Assuming JWT claims are always fresh.

---

## 37. Interview questions

### Q: What is an access token?
A short-lived credential presented to protected APIs after authentication. It can be a signed JWT or an opaque token.

### Q: Why do we need it?
So the user doesn't need to send their password again on every API request.

### Q: Why short-lived?
If a bearer token is stolen, a shorter lifetime reduces the period it can be reused.

### Q: Where should a React app store it?
Options include **memory**, **HttpOnly cookie**, or **localStorage**, each with trade-offs. Memory + HttpOnly refresh cookie is a useful SPA pattern when Bearer headers are required. Avoid assuming localStorage is secure against XSS.

### Q: Can HttpOnly cookies eliminate XSS?
No. They prevent JavaScript from directly reading that cookie, but malicious scripts may still make authenticated requests.

### Q: What happens when access token expires?
API generally returns **401**; client may use a valid refresh credential to obtain a new access token, otherwise login again.

### Q: Is an access token always JWT?
No. Opaque tokens also exist.

### Q: Access token vs refresh token?
Access token accesses APIs; refresh token obtains replacement access tokens.

### Q: Does valid token mean permission to access any resource?
No. Authentication and authorization are separate.

### Q: Best practice for React + Express?
Use HTTPS, short access-token TTL, verify signature and claims, choose storage based on risks, protect refresh tokens, and check permissions server-side.
---

## 38. Strong interview answer

> An access token is a credential presented on protected API requests. I usually keep access tokens short-lived because they are bearer credentials and therefore valuable if stolen. If JWT is used, I verify signature, expiration, issuer, audience, and allowed algorithm, and keep claims minimal. Access tokens should never be treated as permanent sessions; long-lived continuity belongs to a separate refresh-token or session mechanism.

---

## Interview-Ready Summary

```text
Access Token
   |
   +--> short-lived
   +--> API credential
   +--> bearer token
   +--> minimal claims
   +--> verify every request

Never:
long-lived unnecessarily
log token
trust decode-only claims
store secrets in payload
```

## Practical Task

Design a protected API where:

- access token lasts 15 minutes
- token contains sub, role, iss, aud, exp, jti
- admin routes require fresh DB role validation
- expired token triggers refresh flow
