# Lesson 145 — Brute-force and Credential-Stuffing Protection

## 1. What is a brute-force attack?

A **brute-force attack** repeatedly tries guesses until a password or code is found.

A **dictionary attack** tries common passwords. **Credential stuffing** tries email/password pairs leaked from other websites. **Password spraying** tries one common password against many accounts to avoid per-account lockouts.

Example: An attacker repeatedly sends requests to `POST /auth/login`. The backend must prevent fast guessing without making login impossible for real users.

## 2. Attack flow

```text
Attacker -> many login attempts
             |
             v
       Authentication API
             |
             +-- no protection -> many guesses / DB + hash load
             |
             +-- protection -> slow, reject, alert, require step-up
```

## 3. Defense in layers

1. **Strong password hashing:** use Argon2id (preferred where suitable) or properly configured bcrypt/scrypt. Never store plaintext passwords.
2. **IP-based rate limiting:** slows attempts from one source.
3. **Account-based rate limiting:** slows attacks distributed across many IPs.
4. **MFA / passkeys:** protect even if a password is compromised.
5. **Generic login errors:** do not reveal whether an email exists.
6. **Monitoring:** detect unusual failed logins and credential-stuffing patterns.
7. **Safe recovery:** protect password-reset endpoints too.

Avoid permanent account lockouts based only on failures: attackers can intentionally lock out victims.

## 4. Express 5 example: IP-based login limiter

```bash
npm install express express-rate-limit bcryptjs
```

```js
import { rateLimit } from "express-rate-limit";

const loginIpLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 10,
  standardHeaders: "draft-8",
  legacyHeaders: false,
});

app.post("/auth/login", loginIpLimiter, loginController);
```

This is a **first layer**, not a complete defense: an attacker may use multiple IP addresses.

## 5. Account-based attempt tracking (concept)

Use a **shared Redis store** keyed by a privacy-preserving identifier derived from the normalized login identifier, rather than exposing raw emails in keys/logs.

```text
POST /auth/login { email, password }
    |
    v
Normalize and validate email
    |
    v
Check account-attempt counter (shared Redis)
    |
    +-- threshold exceeded -> generic 429 / temporary delay
    |
    v
Check password
    |
    +-- incorrect -> increment failed-attempt counter
    |
    +-- correct -> reset relevant counter, issue session
```

A counter needs an **expiry (TTL)**, and increments must be atomic in the shared store. Set thresholds and cool-downs carefully to avoid attackers causing easy denial of service against another user's account.

## 6. Real login implementation: verify password correctly

Assume the registration flow already stores `passwordHash` and your `User` model has `email` and `passwordHash` fields.

```js
import bcrypt from "bcryptjs";

async function loginController(req, res) {
  const { email, password } = req.body; // Validate types/lengths first.

  const user = await User.findOne({ email })
    .select("+passwordHash");

  const valid = user
    ? await bcrypt.compare(password, user.passwordHash)
    : false;

  if (!valid) {
    return res.status(401).json({
      message: "Invalid email or password",
    });
  }

  // Create a secure session using your auth/session layer.
  return res.json({ message: "Login successful" });
}
```

**Security note:** This code is an educational core only. Production systems should also consider response timing differences for unknown users, session creation/rotation, MFA, account-level rate limits, and monitoring. Never return the password hash.

## 7. Password hashing vs encryption

- **Hashing:** one-way verification. You compare a submitted password to a stored hash.
- **Encryption:** reversible with a key. Not the normal choice for password storage.
- **Salt:** random value included in a password hash; modern password hashing libraries handle it.
- **Work factor:** controls how expensive guessing becomes. Choose settings based on measured server capacity.

For registration:

```js
const passwordHash = await bcrypt.hash(password, 12);
await User.create({ email, passwordHash });
```

Avoid hashing the same password twice (for example, in both a controller and a Mongoose `pre("save")` hook). Choose one clear place.

## 8. MFA, passkeys, and recovery

- **MFA:** requires another proof besides a password (for example, authenticator-based one-time codes).
- **Passkeys:** use public-key authentication and are phishing-resistant when implemented correctly.
- **Recovery codes:** generate securely, store protected/hashed as appropriate, and make single-use.
- **Password reset:** use cryptographically random, short-lived, single-use tokens; do not disclose whether an account exists.

Rate limit OTP verification and password-reset requests too. Never log OTPs or reset tokens.

## 9. Login response design and account enumeration

Bad:

```text
"No account with this email"
"Correct email, wrong password"
```

Better:

```text
"Invalid email or password"
```

But text alone isn't enough: different HTTP codes, timing, response lengths, or password-reset messages may still reveal account existence. Test for consistency.

## 10. Handling repeated failures

Use escalating temporary delays or additional verification rather than unlimited retries or indefinite lockouts. For a suspicious login, you might:
- require a passkey/MFA challenge,
- temporarily slow further attempts,
- notify the account owner without exposing details,
- record a safe security event for investigation.

Make sure a single attacker cannot trivially lock out every account by guessing emails.

## 11. Common mistakes

1. Only limiting per IP, ignoring distributed attacks.
2. Saving plaintext or reversibly encrypted passwords.
3. Returning different messages for existing and missing users.
4. Forgetting OTP, reset, and registration abuse controls.
5. Creating unlimited expensive password-hash operations under load.
6. Logging credentials or reset tokens.
7. Relying on CAPTCHA alone instead of layered protections.

## 12. Practice

1. Add IP-based rate limiting to a login route.
2. Design a Redis key/TTL strategy for account-based failures.
3. Explain the difference between brute force, password spraying, and credential stuffing.
4. Test generic error responses for wrong email and wrong password.
5. Explain why MFA protects accounts even if passwords are reused.

**Interview answer:** Brute-force protection combines password hashing, IP and account-level rate limits, MFA/passkeys, safe recovery, consistent errors, monitoring, and temporary progressive defenses. No single control stops all attacks.
