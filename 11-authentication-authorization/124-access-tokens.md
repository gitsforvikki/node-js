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

An access token is a credential presented by a client to access protected resources.

Example:

```http
Authorization: Bearer <access-token>
```

The server verifies the token and decides whether the caller is authenticated.

---

## 2. Access token purpose

After login:

```text
email + password
      |
      v
credentials verified
      |
      v
access token issued
      |
      v
client uses token on future API calls
```

The user does not need to submit the password again for every request.

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

Suppose an attacker steals a token.

If expiration is:

```text
30 days
```

the attacker may have access for a long time.

If expiration is:

```text
15 minutes
```

the exposure window is much smaller.

---

## 8. Typical lifetime

Common examples:

```text
5 minutes
15 minutes
30 minutes
1 hour
```

Exact value depends on:

- app sensitivity
- client type
- refresh-token design
- risk tolerance

There is no single universally correct TTL.

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

## 18. Access token storage in browser

Possible approaches:

### Memory

```text
JavaScript memory
```

Benefits:
- disappears on reload

Trade-off:
- requires refresh flow after reload

### HttpOnly cookie

Benefits:
- not readable by JavaScript

Trade-off:
- browser auto-sends cookie
- CSRF must be considered

### localStorage

Convenient but JavaScript-readable.

If XSS occurs:

```text
attacker script
   |
   v
localStorage token
   |
   v
token theft
```

---

## 19. Access token in Authorization header

Example:

```http
Authorization: Bearer eyJ...
```

Common for:

- SPAs
- mobile apps
- API clients
- service-to-service requests

---

## 20. Access token in cookie

Example:

```http
Cookie: accessToken=...
```

Common for browser apps.

If cookie is HttpOnly:

```text
JavaScript cannot read token
```

This reduces token theft from many XSS scenarios.

---

## 21. Cookie-based access token and CSRF

Because browsers automatically attach cookies, malicious sites may cause unwanted authenticated requests.

Defenses include:

- SameSite
- CSRF token
- origin checks
- correct CORS policy

---

## 22. Authorization header and CSRF

A custom Authorization header is not automatically sent by browser to arbitrary sites.

That makes classic CSRF less likely.

But if token is stored in JavaScript-accessible storage, XSS risk increases.

This is a trade-off.

---

## 23. Access token verification middleware

Example concept:

```js
async function authenticate(
  req,
  res,
  next
) {
  try {
    const header =
      req.headers.authorization;

    if (
      !header?.startsWith(
        "Bearer "
      )
    ) {
      return res
        .status(401)
        .json({
          error: {
            code:
              "AUTH_REQUIRED",
          },
        });
    }

    const token =
      header.slice(7);

    const payload =
      await verifyAccessToken(
        token
      );

    req.auth = {
      userId:
        payload.sub,
      role:
        payload.role,
    };

    next();
  } catch (error) {
    next(error);
  }
}
```

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

## 25. Expired token

Response usually:

```text
401 Unauthorized
```

Client may then try refresh flow.

---

## 26. Invalid token

Examples:

- bad signature
- malformed token
- wrong issuer
- wrong audience
- unsupported algorithm

Treat as authentication failure.

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

```text
Access Token
  short-lived
  sent frequently
  accesses API

Refresh Token
  long-lived
  sent rarely
  obtains new access token
```

This separation reduces risk.

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

### What is an access token?

A short-lived credential used to authorize API requests after authentication.

### Why short-lived?

To reduce the damage window if stolen.

### Is an access token always JWT?

No.

### Where should it be stored?

Depends on client architecture; browser storage choices involve XSS vs CSRF trade-offs.

### What happens when it expires?

Client may use a valid refresh token to obtain a new access token.

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
