# Lesson 121 — Password Hashing

## Why password hashing is critical

Passwords are one of the highest-risk pieces of data in an authentication system.

A production backend should **never store plaintext passwords**.

If your database leaks and passwords are plaintext, every user credential is immediately exposed.

Password hashing reduces the damage of a database breach.

---

## 1. Hashing vs encryption

Before learning password hashing, understand three different jobs:

- **Hashing:** Turn a password into a one-way value so the server can verify it later without recovering the original password.
- **Encryption:** Protect data using a key. An authorized application can decrypt it when it needs the original value.
- **JWT (JSON Web Token):** A token format that can carry verifiable claims. A signed JWT is often used after login to authenticate API requests. JWT is **not** a password hashing algorithm.

### 1.1 The problem with storing plain passwords

Suppose a user registers with:

```js
const password = "Vikash@123";
```

**Bad database record:**

```json
{
  "email": "user@example.com",
  "password": "Vikash@123"
}
```

If the database leaks, the attacker immediately sees the original password. Instead, save a **password hash**.

```text
User password: Vikash@123
        |
        v
Password-hashing algorithm (Argon2id / bcrypt)
        |
        v
Encoded hash: $argon2id$...
        |
        v
Database (store hash, not original password)
```

A password hash is **one-way**: there is no normal decrypt operation to get the password back. This does not mean weak passwords are impossible to guess; attackers can still try candidate passwords against stolen hashes.

### 1.2 Node.js example: hash and verify using bcrypt

Install with `npm install bcrypt`.

**At registration:**

```js
import bcrypt from "bcrypt";

const password = "Vikash@123";
const hashedPassword = await bcrypt.hash(password, 12);

// Save hashedPassword, NOT password, in your database.
```

**At login:**

```js
const isValid = await bcrypt.compare(
  "Vikash@123",
  user.passwordHash
);

if (!isValid) {
  throw new Error("Invalid credentials");
}
```

`bcrypt.compare()` **does not decrypt** the database hash. It uses the salt and cost parameters stored with the encoded hash to verify the password. Real login code must also handle a user that does not exist and should return a generic credentials error.

**Which algorithm?** Prefer **Argon2id** for new systems where supported. **bcrypt** remains a widely used, established option. **SHA-256 alone is not suitable for password storage** because it is too fast; see Sections 5–11 for salt, algorithm and work-factor details.

### 1.3 Encryption: when we need the original data again

Imagine your backend stores an API credential that it must send to another service later. It needs to recover the original value securely, so encryption is appropriate.

```text
Original data
     |
     v
Encrypt using key
     |
     v
Unreadable ciphertext (stored)
     |
     v
Decrypt with authorized key
     |
     v
Original data again
```

For **symmetric encryption**, the same secret key encrypts and decrypts. For **asymmetric encryption**, the key pair has different roles. Use authenticated encryption such as **AES-256-GCM** through well-tested libraries; protect and rotate encryption keys appropriately.

Examples of when encryption is useful:
- Third-party API credentials that the backend must retrieve
- Confidential documents an authorized user will read again
- Sensitive data the application genuinely needs in original form

**Why not encrypt login passwords?** Login only needs to verify a candidate password. If the password were encrypted, the server would hold a key capable of revealing every stored password. A one-way password hash avoids that unnecessary ability.

### 1.4 Hashing vs encryption: quick comparison

| Question | Password hashing | Encryption |
| --- | --- | --- |
| Can we recover original data? | No normal reverse/decrypt operation | Yes, with the required key |
| Needs a reversible secret key? | No (optional separate pepper is different) | Yes / appropriate key pair |
| Main purpose | Verify a password safely | Keep recoverable data confidential |
| Common algorithms | Argon2id, bcrypt, scrypt, PBKDF2 | AES-GCM |
| Store login passwords? | **Yes** | **Generally no** |
| Typical use | Signup + login verification | Protecting recoverable secrets |

### 1.5 Where JWT fits after login

Now imagine the password is verified. The user requests `/profile` or `/orders` later. Asking for email and password on every request is a bad approach. One option is **JWT-based authentication**:

```text
1. Client -> POST /login with email + password
2. Server -> retrieve saved password hash; verify password
3. Server -> issue a short-lived signed JWT
4. Client -> GET /profile, Authorization: Bearer <token>
5. Server -> verify JWT signature and expiry; check authorization
6. Server -> return allowed resource
```

**Minimal Node.js illustration:**

```js
import jwt from "jsonwebtoken";

// Only AFTER successfully verifying the user's password:
const token = jwt.sign(
  { sub: String(user.id) },
  process.env.JWT_SECRET,
  { algorithm: "HS256", expiresIn: "15m" }
);

// In authentication middleware, after extracting the Bearer token:
const payload = jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ["HS256"],
});

console.log(payload.sub); // user ID (subject)
```

Here, `sub` identifies the user; `JWT_SECRET` is a private server-side signing secret; `expiresIn` limits token lifetime. Production systems should also validate relevant claims such as issuer/audience, handle invalid tokens, and check permissions for each protected operation.

**Is JWT encrypted? Usually not.** A typical signed JWT has three parts:

```text
header.payload.signature
```

- **Header:** token metadata / signing algorithm
- **Payload:** claims such as user ID and expiry; Base64URL encoded, **not secret**
- **Signature:** protects integrity and helps verify who issued it, given the trusted signing key

Anyone holding a normal signed JWT can decode its header and payload. **Never store passwords, API secrets, or sensitive personal data in the JWT payload.** Encrypted JWTs (JWE) also exist, but signing does not imply encryption.

JWT is **not mandatory** for authentication. Server-side sessions are another valid approach. JWT verification does not automatically mean a user has permission to perform a particular action: **authentication** confirms identity; **authorization** checks permissions. Token storage, expiration, revocation and logout need deliberate design.

### 1.6 When to use what: real backend scenarios

| Backend requirement | Best-fit mechanism |
| --- | --- |
| Store password at signup | Password hashing |
| Check password at login | Hash verification |
| Securely store recoverable third-party credentials | Encryption |
| Authenticate later API requests | Signed JWT **or** server-side session |
| Allow access to another user's order? | Authorization check after authentication |

### 1.7 Interview questions

**Q: Why hash passwords instead of encrypting them?**  
Because login only needs to verify a submitted password, not retrieve the original. Reversible encryption creates an unnecessary key that could expose stored passwords.

**Q: Can hackers crack password hashes?**  
They may guess weak passwords offline if they steal hashes. Unique salts and deliberately expensive password-hashing algorithms make guessing harder; they cannot guarantee absolute safety.

**Q: Does JWT hash passwords?**  
No. Password hashing protects stored credentials; JWT carries signed or otherwise protected claims for subsequent requests.

**Q: Is JWT payload secure from readers?**  
No. A common signed JWT is encoded, not encrypted. Do not place confidential data there.

**Q: Does verifying a JWT automatically authorize a user?**  
No. Your application must still check access to the requested resource.

**Remember:** **Hash the password. Encrypt data you must recover. Use JWT or sessions to maintain authentication, then check authorization.**


---

## 2. Why one-way matters

Authentication does not need to recover the original password.

It only needs to answer:

```text
Does this candidate password match the stored password?
```

So the system should not need the original password after registration.

---

## 3. Registration flow

```text
User enters password
      |
      v
Validate password policy
      |
      v
Password hashing function
      |
      v
Store hash
```

Database stores:

```text
$2b$...
```

not:

```text
MyPassword123
```

---

## 4. Login flow

```text
candidate password
      |
      v
password verification
      |
      +--> match
      |      |
      |      v
      |   authenticate
      |
      +--> no match
             |
             v
          reject
```

You do not hash with a random new salt manually and compare strings unless the password library's verification algorithm handles the stored hash format.

---

## 5. Why SHA-256 alone is not enough

A common mistake:

```js
sha256(password)
```

SHA-256 is designed to be fast.

For password storage, fast is bad.

Attackers can try billions of guesses much more efficiently with fast hashes.

Password hashing algorithms are intentionally expensive.

---

## 6. Password hashing algorithms

Common password-specific algorithms include:

- Argon2
- scrypt
- bcrypt
- PBKDF2

They are designed to make brute-force attacks more expensive.

---

## 7. Salt

A salt is a random value combined with the password before hashing.

Purpose:

- same passwords produce different hashes
- prevents easy rainbow-table reuse
- makes precomputed attacks less useful

Example:

```text
User A password: hello123
User B password: hello123

with unique salts:

hash A != hash B
```

---

## 8. Salt is not secret

Important interview point:

> A salt does not need to be hidden.

It is typically stored alongside the hash.

Its job is uniqueness, not secrecy.

---

## 9. Pepper

A pepper is an additional secret value stored separately from the database.

Example:

```text
password
+
salt
+
server-side pepper
```

Pepper may add defense if DB alone is compromised.

But it introduces:
- secret management
- rotation complexity

It is optional and architecture-dependent.

---

## 10. Work factor

Password hashing should be deliberately expensive.

Concept:

```text
higher cost
   |
   +--> slower login hash
   +--> slower attacker guesses
```

Choose cost so legitimate requests remain acceptable while attacks become expensive.

---

## 11. Cost tuning

Do not copy a cost forever from a tutorial.

Benchmark on your production-like hardware.

Example goal:

```text
hash time:
roughly tens/hundreds of milliseconds
```

Exact values depend on:
- hardware
- traffic
- security requirements

---

## 12. Security vs availability

If hashing is too expensive:

```text
many login attempts
   |
   v
CPU saturation
   |
   v
Node app slows down
```

So password security must be combined with:

- rate limiting
- login throttling
- MFA
- monitoring

---

## 13. Node.js and CPU-bound hashing

Hashing is CPU-intensive.

In Node.js, many password libraries use native implementations / libuv thread pool behavior.

You should understand that huge concurrent hashing workloads can still reduce application throughput.

---

## 14. Registration example

```js
const passwordHash =
  await hashPassword(
    req.body.password
  );

await userRepository.create({
  email:
    req.body.email,
  passwordHash,
});
```

Only store the hash.

---

## 15. Login example

```js
const user =
  await userRepository
    .findByEmailWithPassword(
      email
    );

const matches =
  await verifyPassword(
    password,
    user.passwordHash
  );

if (!matches) {
  throw new InvalidCredentialsError();
}
```

---

## 16. Generic login errors

Avoid:

```text
Email does not exist
```

and:

```text
Password incorrect
```

as separate public responses.

Safer:

```text
Invalid credentials
```

This reduces account enumeration.

---

## 17. Password policy

A password policy may include:

- minimum length
- breached-password checks
- MFA encouragement
- password manager friendliness

Avoid forcing overly complex arbitrary rules that lead users to predictable patterns.

Length is generally more useful than requiring many symbol categories.

---

## 18. Maximum password length

Do not accept unbounded password bodies.

Even password hashing endpoints should have reasonable input limits to prevent resource abuse.

---

## 19. Rehashing strategy

Security settings improve over time.

Suppose stored password uses old cost:

```text
cost 10
```

new policy:

```text
cost 12
```

On successful login:

```text
verify old hash
   |
   v
rehash with new cost
   |
   v
update DB
```

This is transparent migration.

---

## 20. Password reset

Never email existing password.

Correct reset flow:

```text
user requests reset
   |
   v
generate random one-time token
   |
   v
store secure token representation + expiry
   |
   v
send reset link
   |
   v
verify token
   |
   v
set new password hash
   |
   v
invalidate reset token
```

---

## 21. Password reset token storage

Do not necessarily store the raw reset token.

Better:

```text
raw token -> user
hash(token) -> DB
```

If DB leaks, active reset tokens are less directly usable.

---

## 22. Changing password

A secure change-password flow often requires:

- authenticated user
- verify current password
- validate new password
- hash new password
- invalidate old refresh sessions where appropriate

---

## 23. Do not log passwords

Bad:

```js
console.log(req.body);
```

on login route can expose plaintext password in logs.

Redact sensitive fields.

---

## 24. Do not return hashes

Never include:

```text
passwordHash
```

in:
- API responses
- logs
- analytics
- error messages

---

## 25. Timing attacks

Password verification libraries are designed to avoid simple direct string comparison pitfalls.

Do not implement password verification using naive:

```js
storedHash ===
hash(candidate)
```

with home-grown crypto design.

Use mature password libraries.

---

## 26. Database breach threat model

If attacker steals DB:

Stored:

```text
email
passwordHash
salt
```

They can still perform offline guessing.

Password hashing does not make cracking impossible.

It makes guesses expensive.

---

## 27. Credential stuffing

Hashing does not protect against attackers using credentials leaked from another service.

Defenses include:

- MFA
- breached-password detection
- login rate limits
- anomaly detection

---

## 28. Common mistakes

### Mistake 1
Plaintext storage.

### Mistake 2
Encrypting passwords reversibly.

### Mistake 3
SHA-256 alone.

### Mistake 4
Logging plaintext passwords.

### Mistake 5
No rate limiting around CPU-heavy verification.

### Mistake 6
Returning password hashes.

### Mistake 7
No rehash strategy.

---

## 29. Interview questions

### Hashing vs encryption?

Hashing is one-way; encryption is reversible with a key. Passwords should usually be hashed.

### Why not SHA-256?

It is too fast and therefore efficient for brute-force attackers.

### What is a salt?

A unique random value that ensures identical passwords produce different hashes.

### Is salt secret?

No.

### What is a pepper?

An optional server-side secret added to password processing and stored separately from the DB.

---

## 30. Strong interview answer

> Passwords should be stored using a dedicated slow password-hashing algorithm such as Argon2, scrypt, or bcrypt, never plaintext or reversible encryption. Each password should have a unique salt, and the work factor should be tuned to production hardware. During login, I verify the candidate password with the library's comparison function, return generic credential errors, rate-limit attempts, never log plaintext passwords, and rehash successfully authenticated passwords when security parameters become outdated.

---

## Interview-Ready Summary

```text
Password
   |
   v
slow password hash
   |
   v
salted hash in DB

Never:
plaintext
reversible encryption
fast generic hashes
logging passwords

Remember:
salt != secret
pepper = optional secret
cost needs tuning
```

## Practical Task

Design:

- signup
- login
- change password
- forgot password
- reset password

Show exactly where plaintext exists temporarily and where hashes are stored.
