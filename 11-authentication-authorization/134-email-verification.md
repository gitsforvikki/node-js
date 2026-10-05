# Lesson 134 — Email Verification

## Why email verification matters

Email verification proves that the user controls the email address they registered.

It helps reduce:

- fake accounts
- mistyped addresses
- spam
- abuse
- account-recovery problems

But verification flow must itself be secure.

---

# 1. Signup Flow

```text
User signs up
   |
   v
Account created
   |
   v
emailVerified = false
   |
   v
Generate verification token
   |
   v
Send verification email
   |
   v
User clicks link
   |
   v
Verify token
   |
   v
emailVerified = true
```

---

# 2. User Model

Example:

```json
{
  "id": "usr_123",
  "email": "vikash@example.com",
  "emailVerified": false
}
```

or timestamp:

```json
{
  "emailVerifiedAt": null
}
```

Timestamp is often more useful for auditing.

---

# 3. Verification Token

Generate high-entropy random token.

Example:

```js
crypto.randomBytes(32)
```

Properties:

- random
- single-use
- purpose-specific
- expiring

---

# 4. Store Hash, Not Raw Token

Same principle as password reset:

```text
raw token
  -> email

SHA-256(token)
  -> DB
```

---

# 5. Verification Record

Example:

```text
email_verification_tokens

id
user_id
email
token_hash
expires_at
used_at
created_at
```

Storing email in token record is helpful if users can change email before verification.

---

# 6. Verification Link

Example:

```text
https://app.example.com/verify-email?token=<raw-token>
```

Use HTTPS.

---

# 7. Verify Endpoint

Possible:

```text
POST /auth/verify-email
```

Body:

```json
{
  "token": "..."
}
```

Using POST can avoid state-changing behavior from link-preview bots that automatically perform GET requests.

A frontend verification page can read query token and submit POST.

---

# 8. Link Scanner Problem

Email providers and security tools sometimes pre-open links.

If verification happens immediately on:

```text
GET /verify?token=...
```

a scanner may consume the token before the user clicks.

Safer pattern:

```text
GET opens confirmation page
POST performs verification
```

This is an important production consideration.

---

# 9. Atomic Verification

Two requests may attempt to use token.

Use atomic consume:

```text
token unused
AND
not expired
```

Then mark used and update email verification state.

---

# 10. Token Expiration

Typical expiry:

```text
15 minutes
1 hour
24 hours
```

Verification tokens can usually live longer than password-reset tokens because risk profile differs, but keep finite.

---

# 11. Resend Verification

Endpoint:

```text
POST /auth/resend-verification
```

Should be rate limited.

Prevent email spam.

---

# 12. Invalidate Previous Tokens

When resending:

Option:

```text
invalidate all previous unused tokens
```

Then only latest token works.

This simplifies reasoning.

---

# 13. Verify Current Email

Important edge case.

User signs up with:

```text
old@example.com
```

then changes email to:

```text
new@example.com
```

Old verification token must not verify the new email accidentally.

Token should be bound to specific email value.

---

# 14. Email Change Flow

For changing email:

```text
authenticated user requests new email
      |
      v
send verification to new email
      |
      v
verify ownership
      |
      v
replace account email
```

Do not mark new email verified just because old email was verified.

---

# 15. Sensitive Actions Before Verification

You may allow:

- login
- profile completion

but block:

- sending messages
- payments
- publishing
- invitations

depending on product requirements.

---

# 16. Middleware

Example:

```js
function requireVerifiedEmail(
  req,
  res,
  next
) {
  if (
    !req.user.emailVerifiedAt
  ) {
    return res
      .status(403)
      .json({
        error: {
          code:
            "EMAIL_NOT_VERIFIED",
        },
      });
  }

  next();
}
```

---

# 17. Verification Is Not Authentication

Email verification proves control of email.

It does not replace:

- login
- password
- MFA
- session

Different security property.

---

# 18. Do Not Put User ID Only in Link

Bad:

```text
/verify-email?userId=123
```

Anyone could verify arbitrary users.

Use unguessable secret token.

---

# 19. Signed JWT Verification Token?

Possible:

```text
JWT containing user/email/exp
```

But you still may want one-time-use state.

Opaque random token + stored hash is simpler for revocation and one-time use.

---

# 20. Token Purpose

Never reuse same token across:

- email verification
- password reset
- login
- email change

Purpose separation prevents confused-deputy bugs.

---

# 21. Enumeration

Resend endpoint can leak whether email exists.

Safer public response:

```text
If the account exists and requires verification, an email will be sent.
```

---

# 22. Rate Limiting

Protect:

- per IP
- per account
- per email address

Examples:

```text
3 emails / hour
```

Exact numbers depend on application.

---

# 23. Email Queue

Send verification email asynchronously.

```text
request
  |
  v
create token
  |
  v
queue email
  |
  v
return response
```

This avoids tying endpoint latency to email provider.

---

# 24. Token Hash Example

```js
const rawToken =
  crypto
    .randomBytes(32)
    .toString("hex");

const tokenHash =
  crypto
    .createHash("sha256")
    .update(rawToken)
    .digest("hex");
```

---

# 25. Verification Service

```js
async function verifyEmail(
  rawToken
) {
  const tokenHash =
    hashToken(
      rawToken
    );

  const token =
    await tokenRepo
      .consumeValidToken(
        tokenHash
      );

  if (!token) {
    throw new InvalidVerificationTokenError();
  }

  await userRepo
    .markEmailVerified({
      userId:
        token.userId,
      email:
        token.email,
    });
}
```

---

# 26. Database Transaction

If token record and user are in same DB:

```text
consume token
+
mark email verified
```

can be in one transaction.

This avoids partial state.

---

# 27. Security Notification

After verification, optionally send:

```text
Your email has been verified.
```

Usually low-risk but useful.

---

# 28. Email Verification vs Magic Link Login

Email verification:

```text
prove ownership once
```

Magic link login:

```text
email token authenticates session
```

Different purposes.

---

# 29. Common Mistakes

### Mistake 1
Verification token never expires.

### Mistake 2
Raw token stored in DB.

### Mistake 3
No rate limit on resend.

### Mistake 4
Old token verifies changed email.

### Mistake 5
State-changing verification GET consumed by link scanner.

### Mistake 6
Same token used for multiple purposes.

---

# 30. Interview Questions

## Why verify email?

To prove control of registered address and reduce abuse/recovery problems.

## Why hash verification tokens?

To reduce damage if token database leaks.

## Why can GET verification be problematic?

Email security scanners may automatically open links and consume one-time tokens.

## Should email verification token equal password reset token?

No. Tokens should be purpose-specific.

---

# 31. Strong Interview Answer

> For email verification I generate a cryptographically random, expiring, single-use token, store only its hash, and bind it to the specific user and email address being verified. I rate-limit resend requests and invalidate older tokens when issuing a new one. In production I avoid completing verification directly on a GET link because email scanners may prefetch links; instead the link opens a page that confirms via POST. Verification state is then updated atomically and can be enforced with middleware for features that require a verified address.

---

# Interview-Ready Summary

```text
Signup
  |
  v
emailVerified=false
  |
  v
random token
  |
  +--> raw -> email
  +--> hash -> DB
  |
  v
user confirms
  |
  v
atomic consume
  |
  v
emailVerified=true

Protect:
expiry
rate limit
purpose
email binding
```

## Practical Task

Implement:

- signup with unverified email
- resend verification
- verification confirmation page
- POST verification endpoint
- requireVerifiedEmail middleware
