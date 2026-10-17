# Lesson 149 — Authentication Security Mistakes

## 1. Authentication vs authorization

**Authentication:** Who is the requester? **Authorization:** Can they access this specific route or record?

A valid token proves an identity only if it was **properly verified**. It does not grant permission to read another user's order.

```text
Request -> verify session/token -> identify user
        -> check role + resource ownership
        -> perform action
```

## 2. Mistake: storing passwords incorrectly

**Never** store plaintext passwords or fast hashes such as SHA-256 alone for password storage. Use a slow password-hashing algorithm: Argon2id (preferred where suitable), scrypt, or correctly configured bcrypt.

```js
import bcrypt from "bcryptjs";

// Registration: hash once
const passwordHash = await bcrypt.hash(password, 12);
await User.create({ email, passwordHash });

// Login: compare against hash
const user = await User.findOne({ email }).select("+passwordHash");
const ok = user && await bcrypt.compare(password, user.passwordHash);
```

Validate passwords, use a measured work factor, and avoid double-hashing in both controller and Mongoose hooks. Never return `passwordHash`.

## 3. Mistake: trusting JWT without verifying it

A JWT typically has three parts: header.payload.signature. The payload is **encoded, not encrypted**; do not put passwords or secrets in it.

**Wrong:** treating `jwt.decode(token)` as authentication. Decoding does not validate the signature.

**Correct:**

```js
import jwt from "jsonwebtoken";

function requireAuth(req, res, next) {
  const header = req.get("authorization");
  const token = header?.startsWith("Bearer ") ? header.slice(7) : null;
  if (!token) return res.sendStatus(401);

  try {
    const payload = jwt.verify(token, process.env.JWT_ACCESS_SECRET, {
      algorithms: ["HS256"],
      issuer: "my-api",
      audience: "my-frontend",
    });
    req.user = { id: payload.sub };
    next();
  } catch {
    res.status(401).json({ message: "Unauthorized" });
  }
}
```

This assumes your **token issuer** sets `sub`, `iss`, and `aud` consistently and your signing key is securely generated and stored. Never accept an algorithm chosen by an untrusted token without enforcing allowed algorithms. For asymmetric signing, verify with the appropriate public key and validate key selection carefully.

## 4. Mistake: missing expiration and revocation

Use short-lived access tokens and a carefully protected refresh/session mechanism. JWTs generally remain valid until expiry unless you implement server-side revocation or another invalidation strategy.

**Refresh token lifecycle:**

```text
Login -> short-lived access token + protected refresh token
  -> access expires
  -> client requests refresh
  -> server validates refresh session
  -> rotate refresh token, issue new access token
  -> revoke session on logout or suspicious reuse
```

Store **hashed refresh tokens** (or a secure opaque token reference) in a server-side session store where feasible. Rotate on use, handle concurrent refresh safely, and detect reuse of invalidated tokens. Use secure cookie flags for browser-based refresh tokens.

## 5. Mistake: unsafe cookies and token storage

```js
res.cookie("session", sessionId, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/",
});
```

- `HttpOnly`: JavaScript cannot read cookie value.
- `Secure`: HTTPS transport.
- `SameSite`: reduces some cross-site request exposure.
- Cookie-based auth still needs CSRF defenses for state-changing requests.
- Storing tokens in `localStorage` exposes them to JavaScript and therefore to XSS if present.

Cookie settings depend on your frontend/backend deployment; cross-site cookies may need `SameSite=None; Secure`, which increases the need for robust CSRF protection.

## 6. Mistake: authentication without authorization

**Bad:**

```js
app.get("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  res.json(order);
});
```

Any logged-in user may guess another order ID.

**Better:**

```js
app.get("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findOne({
    _id: req.params.id,
    userId: req.user.id,
  }).select("items total status");

  if (!order) return res.sendStatus(404);
  res.json(order);
});
```

Validate IDs and apply role checks where appropriate. Filtering by the authenticated owner is an example of **object-level authorization**.

## 7. Mistake: account enumeration and weak login defenses

Returning "Email not registered" for one case and "Wrong password" for another may let attackers discover accounts.

Use generic errors and consistent behavior:

```js
res.status(401).json({ message: "Invalid email or password" });
```

Add IP + account-based rate limiting, monitor failures, use MFA/passkeys where possible, and protect password-reset and OTP endpoints. Avoid permanent lockouts that attackers can trigger against victims.

## 8. Mistake: insecure password reset

A safe password-reset flow:
1. User submits email; return the **same generic response** whether account exists or not.
2. Generate a cryptographically random token with `randomBytes`.
3. Store only a **hash** of the token with an expiration and user reference.
4. Email a link over HTTPS; avoid putting token in analytics/logs.
5. On redemption, verify hash, expiry, and **single-use** status atomically.
6. Update password hash, revoke relevant sessions, and notify user.

Do not use predictable reset codes, permanent reset links, or user-controlled redirect URLs.

## 9. Mistake: session fixation and hijacking

**Session fixation:** attacker causes victim to use a known session identifier before login. **Session hijacking:** attacker steals/reuses a valid session.

Protections:
- Regenerate/rotate session IDs on login and privilege changes.
- Use HTTPS, `HttpOnly`, `Secure`, suitable `SameSite`.
- Set server-side session expiration and revoke on logout.
- Do not expose session IDs in URLs or logs.
- Consider suspicious-session detection and reauthentication for sensitive actions.

## 10. Mistake: leaking secrets or sensitive data

Never log passwords, bearer tokens, refresh tokens, session IDs, or raw reset links. Do not send full Mongoose documents containing hashes/internal fields. Use explicit response DTOs and least-privilege secret storage.

## 11. Secure authentication checklist

- Passwords hashed with a password-specific algorithm.
- JWT signature, algorithm, issuer, audience, and expiry verified.
- Access/refresh tokens have sensible lifetimes and revocation strategy.
- Secure cookie configuration and CSRF protection where relevant.
- Resource-level authorization on every protected operation.
- Rate limits, MFA/passkeys, safe password reset, and monitoring.
- No secrets in source code, logs, URLs, or public frontend bundles.

## 12. Practice

1. Explain why `jwt.decode()` is not enough for authentication.
2. Build a `requireAuth` middleware that verifies issuer and audience.
3. Add owner checks to `GET /orders/:id`.
4. Design a refresh-token rotation flow and explain reuse detection.
5. Explain the difference between session fixation and session hijacking.

**Interview answer:** Secure authentication is more than issuing a JWT. It requires correct password hashing, token/session verification, expiry and revocation, secure cookies, CSRF defenses, brute-force protection, safe recovery, and authorization on each resource.
