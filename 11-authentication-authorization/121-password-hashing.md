# Lesson 121 — Password Hashing

## Why password hashing is critical

Passwords are one of the highest-risk pieces of data in an authentication system.

A production backend should **never store plaintext passwords**.

If your database leaks and passwords are plaintext, every user credential is immediately exposed.

Password hashing reduces the damage of a database breach.

---

## 1. Hashing vs encryption

Hashing:

```text
password
   |
   v
one-way function
   |
   v
hash
```

Encryption:

```text
plaintext
   |
   v
encryption key
   |
   v
ciphertext
   |
   v
decryption key
   |
   v
plaintext
```

Passwords should generally be **hashed**, not reversibly encrypted.

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
