# Lesson 125 — Refresh Tokens

## Why refresh tokens matter

Refresh tokens solve a key authentication problem:

> How can access tokens stay short-lived without forcing the user to log in every 15 minutes?

A refresh token is a long-lived credential used to obtain a new access token.

Because it lives longer, it must be protected much more carefully.

---

## 1. Basic flow

```text
Login
  |
  v
Access Token + Refresh Token
  |
  v
Access Token expires
  |
  v
Client sends Refresh Token
  |
  v
Server validates refresh session
  |
  v
New Access Token
```

---

## 2. Access vs refresh token

```text
Access Token
  short TTL
  sent often
  accesses APIs

Refresh Token
  long TTL
  sent rarely
  only used at refresh endpoint
```

---

## 3. Why not just use one 30-day token?

Because every API request would carry a credential valid for 30 days.

If stolen:

```text
attacker gets long access window
```

Separating tokens reduces exposure.

---

## 4. Refresh token can be opaque

A strong design is:

```text
random 256-bit token
```

Client stores raw token.

Server stores:

```text
hash(refreshToken)
```

This behaves like a session secret.

---

## 5. Refresh token can also be JWT

Possible, but JWT does not remove need for revocation if you want secure logout and rotation.

Many production systems still keep server-side refresh-session state.

---

## 6. Refresh session model

Example DB record:

```text
refresh_sessions

id
user_id
token_hash
expires_at
created_at
revoked_at
replaced_by
user_agent
ip
```

---

## 7. Why store hash instead of raw refresh token?

Suppose DB leaks.

If raw refresh tokens are stored:

```text
attacker can immediately use them
```

If only hashes are stored:

```text
raw token still required
```

This mirrors password/reset-token storage principles.

---

## 8. Refresh endpoint

Example:

```text
POST /auth/refresh
```

Flow:

```text
read refresh token
   |
   v
hash it
   |
   v
find active session
   |
   v
check expiry/revocation
   |
   v
rotate token
   |
   v
issue new access token
```

---

## 9. Refresh token rotation

Rotation means:

> Every time a refresh token is used, invalidate it and issue a new refresh token.

Flow:

```text
RT1 used
  |
  v
RT1 revoked
  |
  v
RT2 issued

RT2 used
  |
  v
RT2 revoked
  |
  v
RT3 issued
```

---

## 10. Why rotation matters

Suppose attacker steals RT1.

User later refreshes and server rotates to RT2.

RT1 becomes invalid.

This limits stolen token lifetime.

---

## 11. Refresh-token reuse detection

Very important advanced concept.

Scenario:

```text
RT1 used legitimately
   |
   v
RT2 issued

Later attacker uses old RT1
```

That suggests token theft.

Server can respond by revoking the whole token family/session chain.

---

## 12. Token family

Conceptually:

```text
Session Family
   |
   +--> RT1
   +--> RT2
   +--> RT3
```

If a revoked ancestor appears again:

```text
possible replay/theft
```

Revoke family.

---

## 13. Rotation race condition

Two browser requests may attempt refresh simultaneously.

```text
Request A uses RT1
Request B uses RT1
```

If both succeed, rotation protection breaks.

Use atomic update/transaction logic.

---

## 14. Atomic rotation concept

```text
UPDATE refresh_session
SET revoked_at = now()
WHERE token_hash = ?
AND revoked_at IS NULL
```

Only one request should succeed.

Then issue replacement.

---

## 15. Expiration

Refresh token lifetime might be:

```text
7 days
30 days
90 days
```

Depends on:
- app sensitivity
- remember-me policy
- device/session management

Longer duration means higher theft risk.

---

## 16. Absolute vs sliding expiration

Absolute:

```text
session expires 30 days after login
no matter how active user is
```

Sliding:

```text
each refresh extends session
```

Sliding sessions need a maximum absolute cap to avoid never-ending sessions.

---

## 17. Logout

Secure logout:

```text
client requests logout
   |
   v
revoke refresh session
   |
   v
clear refresh cookie
```

Access token may still exist briefly until expiration unless explicitly revoked.

---

## 18. Logout all devices

Server can revoke all refresh sessions for user:

```text
UPDATE sessions
SET revoked_at = now()
WHERE user_id = ?
```

Useful after:

- password reset
- security incident
- account recovery

---

## 19. Device sessions

You can show:

```text
Chrome on Linux
Android
iPhone
```

Each device/session has a separate refresh token record.

User can revoke one device independently.

---

## 20. User agent and IP

Useful as metadata:

```text
user_agent
ip
last_used_at
```

But do not rely on IP/user-agent as strong identity.

They can change or be spoofed.

---

## 21. Refresh token storage in browser

Best common browser approach:

```text
HttpOnly
Secure
SameSite
cookie
```

Why?

JavaScript cannot read it directly.

This helps protect the most valuable long-lived token from XSS theft.

---

## 22. Why not localStorage for refresh token?

Refresh token is long-lived.

If XSS steals it:

```text
attacker can keep minting access tokens
```

That is much worse than losing a short-lived access token.

---

## 23. CSRF on refresh endpoint

If refresh token is in cookie, browser automatically sends it.

Protect refresh endpoint with:

- SameSite
- CSRF protection if needed
- Origin checks
- strict CORS

---

## 24. SameSite architecture

SameSite behavior depends on whether frontend/backend are:

- same-site
- cross-site
- separate subdomains

This must be designed deliberately.

---

## 25. Refresh endpoint should be narrow

A refresh token should generally only be accepted at:

```text
/auth/refresh
```

Do not use it to access normal API routes.

---

## 26. Never return refresh token in logs

Do not log:
- request cookies
- refresh token body
- Authorization headers

Redact credentials.

---

## 27. Password change behavior

After password change, decide:

- revoke all refresh sessions
- revoke all except current session
- require login again

For high-security apps, revoke all sessions.

---

## 28. Refresh token theft detection

Signals:

- reuse of old token
- impossible travel
- sudden UA/IP change
- multiple reuse attempts

Advanced systems can trigger:

- session revocation
- user alert
- MFA challenge

---

## 29. Refresh token validation flow

```text
raw refresh token
      |
      v
hash token
      |
      v
find DB session
      |
      +--> missing -> reject
      |
      +--> revoked -> reject / detect reuse
      |
      +--> expired -> reject
      |
      v
rotate atomically
      |
      v
issue access + next refresh token
```

---

## 30. Example refresh service

```js
async function refreshSession(
  rawToken
) {
  const tokenHash =
    hashToken(rawToken);

  const session =
    await sessionRepo
      .consumeActiveToken(
        tokenHash
      );

  if (!session) {
    throw new InvalidRefreshTokenError();
  }

  const nextRefreshToken =
    generateSecureToken();

  await sessionRepo.createReplacement({
    userId:
      session.userId,
    tokenHash:
      hashToken(
        nextRefreshToken
      ),
    parentId:
      session.id,
  });

  const accessToken =
    await signAccessToken({
      sub:
        session.userId,
    });

  return {
    accessToken,
    refreshToken:
      nextRefreshToken,
  };
}
```

---

## 31. Refresh tokens and JWT revocation

Refresh session DB provides server-side control:

```text
logout
password reset
device removal
security incident
```

This is why "JWT means fully stateless" is often a misleading goal.

---

## 32. Access token refresh frequency

Do not refresh on every request.

Refresh only when:

- access token expired
- access token close to expiry
- proactive client strategy requires it

Otherwise refresh endpoint becomes unnecessary load.

---

## 33. Retry behavior

If multiple API calls receive 401 simultaneously, client should avoid triggering ten refresh requests.

Common SPA pattern:

```text
first 401
   |
   v
one refresh promise
   |
   v
other requests wait
   |
   v
retry with new access token
```

This is sometimes called refresh deduplication.

---

## 34. Common mistakes

### Mistake 1
Refresh token never rotates.

### Mistake 2
Store raw refresh tokens in DB.

### Mistake 3
Store long-lived refresh token in localStorage.

### Mistake 4
No logout/revocation state.

### Mistake 5
No reuse detection.

### Mistake 6
Non-atomic rotation.

### Mistake 7
Use refresh token on normal APIs.

---

## 35. Interview questions

### Why use refresh token?

To keep access tokens short-lived without forcing frequent logins.

### Why rotate refresh tokens?

To invalidate previously issued refresh credentials and reduce replay risk.

### What is reuse detection?

Detecting use of an already consumed/revoked refresh token, which can indicate theft.

### Why hash refresh tokens in DB?

To reduce damage if session database leaks.

### Where should browser refresh token live?

Commonly in a Secure HttpOnly cookie.

---

## 36. Strong interview answer

> A refresh token is a long-lived credential used only to obtain new short-lived access tokens. I usually model refresh tokens as server-side sessions: generate a cryptographically random token, store only its hash, keep the raw token in a Secure HttpOnly cookie, and rotate it on every refresh. If an already-used token appears again, I treat it as possible theft and revoke the token family. Rotation must be atomic to prevent concurrent reuse.

---

## Interview-Ready Summary

```text
Refresh Token
   |
   +--> long-lived
   +--> used rarely
   +--> stored securely
   +--> server-side session
   +--> rotated
   +--> revocable

Best practice:
store hash
detect reuse
revoke family
HttpOnly cookie
```

## Practical Task

Design a refresh-token table and implement:

- login
- refresh
- logout
- logout all devices
- token rotation
- reuse detection
