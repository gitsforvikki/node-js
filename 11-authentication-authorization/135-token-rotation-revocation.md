# Lesson 135 — Token Rotation and Revocation

## Why this lesson matters

Issuing tokens is easy.

Securely managing them after issuance is the harder problem.

A strong authentication system needs answers to:

- What happens after logout?
- What if refresh token is stolen?
- What if user changes password?
- How do you revoke all devices?
- How do you detect refresh-token replay?
- How do you rotate signing keys?
- How do you avoid race conditions?

This lesson ties together the entire Section 11.

---

# 1. Token Rotation

Rotation means replacing an existing credential with a new one.

Typical example:

```text
Refresh Token 1
   |
   v
used once
   |
   v
revoked
   |
   v
Refresh Token 2
```

---

# 2. Why Rotate Refresh Tokens?

Without rotation:

```text
stolen refresh token
   |
   v
attacker can reuse until expiry
```

With rotation:

```text
old token becomes invalid after use
```

This reduces replay window.

---

# 3. Token Revocation

Revocation means:

```text
credential was valid
but server now rejects it
```

Reasons:

- logout
- password reset
- user disabled
- device removed
- security incident
- token theft

---

# 4. Access Token Revocation

Short-lived access JWTs are often not stored individually.

Typical strategy:

```text
short TTL
+
refresh-session revocation
```

Access token expires naturally soon.

---

# 5. Immediate Access Revocation

If immediate revocation is required:

Options:

- denylist by jti
- token version
- session record
- user status lookup
- introspection

Trade-off:

You add server-side state.

---

# 6. Denylist

Store revoked token IDs:

```text
jti=abc
expiresAt=...
```

Middleware:

```text
verify JWT
   |
   v
is jti revoked?
   |
   +--> yes -> reject
```

Delete denylist record after token natural expiry.

---

# 7. Denylist Trade-off

If every JWT requires Redis lookup:

```text
fully stateless benefit disappears
```

But security may matter more than architectural purity.

---

# 8. Token Version

User:

```json
{
  "tokenVersion": 12
}
```

JWT:

```json
{
  "sub": "usr_1",
  "ver": 12
}
```

After password reset:

```text
tokenVersion = 13
```

Old tokens fail if middleware checks version.

---

# 9. Session Version

Same idea applies to sessions.

Useful for:

```text
invalidate all sessions
```

without deleting every record individually.

---

# 10. Refresh Token Rotation

Flow:

```text
Client sends RT1
      |
      v
Server validates RT1
      |
      v
Atomically revoke RT1
      |
      v
Create RT2
      |
      v
Return RT2 + new access token
```

---

# 11. Rotation Must Be Atomic

Bad:

```text
check RT1 valid
wait
mark RT1 used
```

Two concurrent requests can both pass check.

Better:

```text
consume-if-unused
```

as one atomic DB operation.

---

# 12. Race Example

```text
Request A -> RT1 valid
Request B -> RT1 valid

A creates RT2
B creates RT3
```

Now two valid descendants exist.

This breaks single-use rotation semantics.

---

# 13. Atomic SQL Pattern

Conceptually:

```sql
UPDATE refresh_tokens
SET used_at = NOW()
WHERE token_hash = $1
AND used_at IS NULL
AND revoked_at IS NULL
AND expires_at > NOW()
RETURNING *;
```

Only one request wins.

---

# 14. Refresh Token Family

Track chain:

```text
RT1
 |
 v
RT2
 |
 v
RT3
```

All belong to same:

```text
family_id
```

---

# 15. Reuse Detection

Suppose RT1 was already used.

Later attacker sends RT1 again.

This indicates:

```text
possible stolen token
```

Action:

```text
revoke entire family
```

Then require reauthentication.

---

# 16. Family Revocation

```text
family_id = abc

RT1 -> revoked
RT2 -> revoked
RT3 -> revoked
```

Useful security response to replay.

---

# 17. Device Session Model

Each login creates:

```text
session family
```

Example:

```text
Chrome laptop -> family A
Android -> family B
```

Revoke one device by revoking one family.

---

# 18. Logout Current Device

```text
revoke current refresh family
clear refresh cookie
```

Short-lived access token expires naturally.

---

# 19. Logout All Devices

```text
revoke all refresh families for user
```

Optionally increment tokenVersion.

---

# 20. Password Reset

Recommended:

```text
change password
   |
   v
revoke all refresh sessions
   |
   v
increment token/session version
```

This removes old authenticated access.

---

# 21. User Disabled

When:

```text
status = disabled
```

middleware should reject.

Also revoke long-lived sessions/tokens.

---

# 22. Key Rotation

JWT signing keys also need rotation.

Example:

```text
Key A
   |
   v
Key B
```

New tokens signed by B.

Old tokens signed by A remain verifiable temporarily.

---

# 23. kid Header

JWT header:

```json
{
  "alg": "RS256",
  "kid": "key-2026-10"
}
```

Verifier selects correct public key using `kid`.

---

# 24. Safe Signing-key Rotation

```text
Step 1
publish new verification key

Step 2
start signing with new key

Step 3
continue accepting old key

Step 4
wait until old tokens expire

Step 5
retire old key
```

Do not immediately delete old verification key while valid tokens still exist.

---

# 25. Secret Rotation for HS256

Harder because same secret signs and verifies.

During migration, verifier may temporarily accept:

```text
old secret
new secret
```

but signer should use new secret.

Asymmetric keys are often easier in distributed systems.

---

# 26. Refresh Token Hashing

Store:

```text
SHA-256(refreshToken)
```

not raw token.

Because token is high entropy, fast hash is acceptable.

---

# 27. Refresh Token Entropy

Use cryptographically random values.

Example:

```text
256-bit random token
```

Never:

```text
userId + timestamp
```

---

# 28. Revocation Table

Possible fields:

```text
id
user_id
family_id
token_hash
parent_id
created_at
expires_at
used_at
revoked_at
replaced_by
user_agent
ip
```

---

# 29. Expired Token Cleanup

Old records should eventually be removed.

Strategies:

- TTL index
- scheduled cleanup
- partition retention
- background job

Do not let revocation table grow forever.

---

# 30. Refresh Token Cookie

Recommended browser storage:

```text
HttpOnly
Secure
SameSite appropriate to topology
```

Do not expose long-lived token to JavaScript unless architecture requires it and risk is understood.

---

# 31. CSRF on Refresh

If token is cookie-based:

Protect refresh endpoint with:

- SameSite
- Origin validation
- CSRF token where needed
- strict CORS

---

# 32. Rotation Grace Window?

Sometimes legitimate clients make concurrent refresh calls.

A tiny grace strategy may be used, but it weakens strict replay detection.

Better client design:

```text
one refresh request at a time
```

Deduplicate refresh calls client-side.

---

# 33. Client Refresh Deduplication

```text
API A -> 401
API B -> 401
API C -> 401
     |
     v
one shared refresh Promise
     |
     v
new access token
     |
     v
retry A/B/C
```

This prevents refresh storms.

---

# 34. Revocation Event Propagation

In distributed services:

```text
Auth Service
  |
  v
revocation event
  |
  +--> Redis pub/sub
  +--> message queue
  +--> shared DB/cache
```

Useful when services cache auth state.

---

# 35. Token Introspection

Instead of local JWT verification:

```text
resource server
   |
   v
authorization server introspection
```

Gets fresh token status.

Trade-off:

- network dependency
- latency

---

# 36. High-risk Action Reauthentication

Even valid token may be insufficient for:

- bank transfer
- password change
- email change
- deleting account

Require:

- recent auth
- MFA
- password confirmation

Token validity does not equal risk approval.

---

# 37. Audit Logs

Log:

```text
session created
refresh rotated
refresh reuse detected
session revoked
all sessions revoked
password reset
signing key rotated
```

Do not log raw tokens.

---

# 38. Security Alerts

On refresh reuse detection:

Possible:

- revoke family
- notify user
- require login
- record IP/device
- trigger MFA

---

# 39. Access Token TTL + Refresh TTL

Example:

```text
Access token:
15 minutes

Refresh session:
7 days

Absolute max:
30 days
```

Exact values depend on product risk.

---

# 40. Sliding Refresh Lifetime

Each valid refresh may extend expiry.

But use maximum absolute expiration.

Without cap:

```text
active session can live forever
```

---

# 41. Common Mistakes

### Mistake 1
Refresh token reusable forever.

### Mistake 2
Rotation not atomic.

### Mistake 3
Raw refresh token stored in DB.

### Mistake 4
No replay/reuse detection.

### Mistake 5
Password reset leaves old sessions active.

### Mistake 6
Immediate deletion of old JWT signing key.

### Mistake 7
No cleanup for expired token records.

### Mistake 8
Logging raw tokens.

---

# 42. Interview Questions

## What is refresh-token rotation?

Replacing each used refresh token with a new one and invalidating the old token.

## What is token reuse detection?

Detecting use of an already consumed refresh token, indicating potential theft.

## How do you revoke JWT?

Short TTL plus denylist, token version, session state, or introspection depending on requirements.

## Why must rotation be atomic?

To prevent multiple concurrent requests from creating multiple valid descendants.

## How do you rotate JWT signing keys?

Publish new verification key, start signing with it, retain old verification key until old tokens expire, then retire old key.

---

# 43. Strong Interview Answer

> I keep access tokens short-lived and manage long-lived authentication through revocable refresh sessions. Every refresh token is a high-entropy random value whose hash is stored server-side. On refresh, I atomically consume the old token and issue a replacement. If an already-used token appears again, I treat that as replay and revoke the entire token family. Logout revokes the current family, password reset can revoke all families and increment a token version, and signing keys are rotated with an overlap period so existing short-lived JWTs can still be verified safely.

---

# Interview-Ready Summary

```text
Access Token
  -> short-lived

Refresh Token
  -> long-lived
  -> stored as hash
  -> rotated on use

Reuse detected
  |
  v
revoke family

Security event
  |
  +--> revoke sessions
  +--> tokenVersion++
  +--> notify user

JWT key rotation:
publish new
sign new
verify old+new
retire old later
```

# Section 11 Final Architecture

```text
Signup
  |
  +--> hash password
  +--> verify email

Login
  |
  v
Authentication
  |
  +--> Session
  |
  +--> JWT Access Token
          |
          v
      short-lived
          |
          v
      Refresh Session
          |
          v
      rotation + revocation

Request
  |
  v
Authentication Middleware
  |
  v
RBAC / Permissions
  |
  v
Ownership / Policy
  |
  v
Controller

Security Events
  |
  +--> password reset
  +--> logout
  +--> account disable
  +--> token theft
          |
          v
      revoke / rotate
```

## Section 11 Complete

This completes the authentication and authorization section from:

```text
Lesson 120
Authentication vs Authorization

through

Lesson 135
Token Rotation and Revocation
```

## Practical Capstone

Build a production authentication module with:

- email/password signup
- bcrypt/Argon2 password hashing
- email verification
- login
- short-lived JWT access token
- hashed rotating refresh token
- Secure HttpOnly refresh cookie
- refresh-token reuse detection
- logout current device
- logout all devices
- RBAC
- object-level authorization
- forgot/reset password
- revoke sessions after password reset
- Google OAuth/OIDC login
- audit logs
