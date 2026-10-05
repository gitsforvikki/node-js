# Lesson 133 — Password Reset Flow

## Why password reset is security-critical

Password reset is effectively an alternate login mechanism.

If an attacker can abuse reset flow:

```text
attacker resets password
   |
   v
owns account
```

Therefore password-reset security should be treated almost as seriously as login itself.

---

# 1. High-level Flow

```text
Forgot Password Request
        |
        v
Generate secure random token
        |
        v
Store token hash + expiry
        |
        v
Email raw token link
        |
        v
User opens reset link
        |
        v
Server verifies token
        |
        v
User submits new password
        |
        v
Hash new password
        |
        v
Invalidate reset token
        |
        v
Invalidate sessions if required
```

---

# 2. Forgot Password Endpoint

Example:

```text
POST /auth/forgot-password
```

Input:

```json
{
  "email": "vikash@example.com"
}
```

---

# 3. Prevent Account Enumeration

Bad:

```text
No account exists with this email.
```

Attacker can discover registered users.

Better:

```text
If an account exists, a reset link has been sent.
```

Return same response whether account exists or not.

---

# 4. Generate Cryptographically Random Token

Use secure randomness.

Conceptually:

```js
crypto.randomBytes(32)
```

Token should be:

- unpredictable
- high entropy
- single-use
- short-lived

---

# 5. Raw Token vs Stored Token

Recommended:

```text
raw token
  -> sent to user

hash(raw token)
  -> stored in DB
```

If database leaks, attacker does not immediately obtain usable reset tokens.

---

# 6. Why Hash Reset Tokens?

Reset tokens behave like temporary passwords.

Storing raw token is equivalent to storing a temporary credential in plaintext.

Hashing reduces breach impact.

---

# 7. Token Record

Example:

```text
password_reset_tokens

id
user_id
token_hash
expires_at
used_at
created_at
```

---

# 8. Expiration

Typical reset link:

```text
10 minutes
15 minutes
30 minutes
1 hour
```

Keep short enough to reduce theft window.

---

# 9. Email Reset Link

Example:

```text
https://app.example.com/reset-password?token=<raw-token>
```

Use HTTPS.

---

# 10. Token in URL Risks

URLs may appear in:

- browser history
- proxy logs
- analytics
- referrer headers

Mitigations:

- short TTL
- one-time use
- avoid third-party content on reset page
- strong Referrer-Policy
- exchange token quickly for a temporary server-side flow if desired

---

# 11. Verify Token

When user submits token:

```text
hash(received token)
   |
   v
lookup matching active record
   |
   v
check expiry
   |
   v
check unused
```

---

# 12. Timing-safe Comparison

If comparing secret-derived values manually, use constant-time comparison primitives.

If DB lookup is performed by hash key, equality lookup is usually sufficient at that layer.

Do not invent custom crypto comparison logic.

---

# 13. One-time Use

After successful reset:

```text
used_at = now
```

or delete record.

Token must never work again.

---

# 14. Atomic Token Consumption

Race:

```text
Request A uses token
Request B uses same token
```

Both arrive simultaneously.

Use atomic update:

```text
UPDATE token
SET used_at = now
WHERE token_hash = ?
AND used_at IS NULL
AND expires_at > now
```

Only one succeeds.

---

# 15. New Password Validation

Before hashing:

- minimum length
- maximum length
- breached password checks if available
- prevent same-as-old password if policy requires

Then hash with secure password algorithm.

---

# 16. Password Hash

Example:

```js
const passwordHash =
  await bcrypt.hash(
    newPassword,
    cost
  );
```

or use Argon2/scrypt based on project standard.

---

# 17. Invalidate Existing Sessions

After reset, often revoke:

- refresh tokens
- active sessions
- remembered devices

Why?

If attacker already had a session, changing password alone may not remove their access.

---

# 18. Token Version Strategy

If using JWT access tokens:

```text
user.tokenVersion++
```

Then old tokens can be rejected if middleware checks version.

---

# 19. Session Strategy

If using sessions:

```text
delete all sessions for user
```

Straightforward immediate revocation.

---

# 20. Email After Reset

Send notification:

```text
Your password was changed.
If this wasn't you, contact support.
```

Do not include the new password.

---

# 21. Rate Limiting

Reset endpoint should be rate limited.

Protect:

```text
POST /forgot-password
```

against:

- email spam
- provider cost abuse
- account enumeration timing
- denial of service

---

# 22. Per-account + Per-IP Limit

Using only IP is not enough.

Attacker can rotate IPs.

Using only account is not enough.

Attacker can target one user.

Combine multiple dimensions.

---

# 23. Timing Leakage

If unknown email returns instantly but known email performs DB + email work, attacker may infer account existence.

Possible mitigation:

- consistent public response
- queue email
- normalize response timing where practical

Do not over-promise perfect timing equality.

---

# 24. Email Sending Should Be Async

Better:

```text
request
  |
  v
create reset token
  |
  v
enqueue email
  |
  v
return generic response
```

Do not make user wait for email provider unnecessarily.

---

# 25. Invalidate Older Reset Tokens

When issuing a new reset request, you may:

- revoke previous active reset tokens
- or allow only the latest token

This reduces confusion and attack surface.

---

# 26. Reset Token Scope

Token should be usable only for:

```text
password reset
```

Do not reuse same token for:

- email verification
- login
- account deletion

Use purpose-specific tokens.

---

# 27. Binding Token to User

Token record should identify target user.

Do not accept userId from client independently and trust it.

The token determines the account.

---

# 28. Password Reset Architecture

```text
POST forgot-password
       |
       v
generic response
       |
       v
secure token
       |
       v
hash token in DB
       |
       v
email raw token
       |
       v
POST reset-password
       |
       v
atomic token consume
       |
       v
hash new password
       |
       v
revoke sessions
```

---

# 29. Example Generate Token

```js
import crypto
  from "node:crypto";

function createResetToken() {
  const rawToken =
    crypto
      .randomBytes(32)
      .toString("hex");

  const tokenHash =
    crypto
      .createHash("sha256")
      .update(rawToken)
      .digest("hex");

  return {
    rawToken,
    tokenHash,
  };
}
```

Using a fast hash for a high-entropy random token is fine because the token is not a human password.

---

# 30. Why SHA-256 Is Fine for Reset Token but Not Password?

Password:

```text
low human entropy
attacker can guess
```

Reset token:

```text
random 256-bit value
practically unguessable
```

Therefore fast cryptographic hash is acceptable for token lookup storage.

This is an excellent interview distinction.

---

# 31. Example Reset Service

```js
async function resetPassword(
  rawToken,
  newPassword
) {
  const tokenHash =
    hashResetToken(
      rawToken
    );

  const record =
    await resetRepo
      .consumeValidToken(
        tokenHash
      );

  if (!record) {
    throw new InvalidResetTokenError();
  }

  const passwordHash =
    await hashPassword(
      newPassword
    );

  await userRepo
    .updatePassword(
      record.userId,
      passwordHash
    );

  await sessionRepo
    .revokeAllForUser(
      record.userId
    );
}
```

---

# 32. Transaction Consideration

Ideally:

```text
consume token
+
update password
+
revoke auth sessions
```

should be coordinated carefully.

If same DB supports transaction, use one where appropriate.

External email notification can occur after commit.

---

# 33. Do Not Auto-login Blindly

Some apps automatically log user in after reset.

This may be okay, but for high-security apps:

```text
reset success
  |
  v
require normal login
```

can be safer.

Choose intentionally.

---

# 34. Common Mistakes

### Mistake 1
Store raw reset token.

### Mistake 2
No expiry.

### Mistake 3
Token reusable.

### Mistake 4
Reveal whether email exists.

### Mistake 5
No rate limiting.

### Mistake 6
Do not revoke old sessions.

### Mistake 7
Predictable token.

### Mistake 8
Reset token used for multiple purposes.

---

# 35. Interview Questions

## Why hash password-reset tokens?

To prevent a database leak from exposing directly usable reset credentials.

## Why can SHA-256 be used for token hash but not password hash?

Reset token has high random entropy, while passwords are guessable and need a slow password-hashing algorithm.

## Should you reveal whether email exists?

Usually no.

## Should password reset invalidate sessions?

Usually yes, especially in security-sensitive applications.

---

# 36. Strong Interview Answer

> Password reset is an alternate authentication path, so I treat reset tokens like temporary secrets. I generate a cryptographically random one-time token, store only a hash with an expiry, send the raw token over HTTPS via email, and consume it atomically during reset. I return the same forgot-password response whether the account exists or not, rate-limit the endpoint, hash the new password with the application's password algorithm, and revoke existing sessions or refresh tokens after reset.

---

# Interview-Ready Summary

```text
Forgot Password
   |
   v
Random Token
   |
   +--> raw token -> email
   +--> hash -> DB
   |
   v
Verify + consume once
   |
   v
Hash new password
   |
   v
Revoke sessions

Security:
short expiry
generic response
rate limit
atomic use
```

## Practical Task

Implement:

- POST /auth/forgot-password
- POST /auth/reset-password

Requirements:

- generic forgot response
- 15-minute token TTL
- hash stored in DB
- one-time atomic use
- revoke all sessions after reset
