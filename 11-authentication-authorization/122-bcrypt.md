# Lesson 122 — bcrypt

## Why bcrypt matters

bcrypt is one of the most widely used password-hashing algorithms in Node.js applications.

Interviewers commonly ask:

- What does bcrypt do?
- What is salt?
- What are salt rounds?
- Why is bcrypt slow?
- How does compare work?
- Is bcrypt encryption?
- What are its practical limitations?

---

## 1. What is bcrypt?

bcrypt is a password hashing algorithm designed to be computationally expensive.

It is:

```text
one-way
salted
cost-configurable
password-specific
```

It is **not encryption**.

---

## 2. Install

```bash
npm install bcrypt
```

Example:

```js
import bcrypt
  from "bcrypt";
```

---

## 3. Hash password

```js
const hash =
  await bcrypt.hash(
    password,
    12
  );
```

The second parameter is commonly called:

```text
salt rounds
cost factor
```

---

## 4. Compare password

```js
const matches =
  await bcrypt.compare(
    candidatePassword,
    storedHash
  );
```

Do not manually extract salt and build your own comparison logic.

---

## 5. What is inside a bcrypt hash?

Example shape:

```text
$2b$12$...
```

Conceptually it contains:

```text
algorithm/version
cost
salt
derived hash
```

That is why one stored bcrypt string is enough for later verification.

---

## 6. bcrypt automatically handles salt

With:

```js
bcrypt.hash(
  password,
  12
)
```

bcrypt generates a random salt automatically.

You usually do not need to generate the salt separately.

---

## 7. Manual salt generation

Possible:

```js
const salt =
  await bcrypt.genSalt(12);

const hash =
  await bcrypt.hash(
    password,
    salt
  );
```

Equivalent idea, but usually unnecessary unless you specifically need control.

---

## 8. Why two users with same password get different hashes

User A:

```text
Password123
```

User B:

```text
Password123
```

Because their salts differ:

```text
hash A != hash B
```

This is expected.

---

## 9. What are salt rounds?

The cost determines how much work bcrypt performs.

Conceptually:

```text
cost 10
   < cost 11
   < cost 12
```

The cost scales exponentially.

Raising cost by 1 approximately doubles the work.

---

## 10. Why cost matters

Low cost:

```text
fast login
fast attacker guesses
```

High cost:

```text
slower login
slower attacker guesses
```

Goal:

> Make password guessing expensive while keeping legitimate authentication usable.

---

## 11. Benchmark rather than guess

Example benchmark:

```js
const start =
  performance.now();

await bcrypt.hash(
  password,
  12
);

console.log(
  performance.now() -
    start
);
```

Benchmark on production-like hardware.

---

## 12. Async vs sync API

bcrypt may offer:

```js
bcrypt.hash()
bcrypt.compare()
```

and synchronous variants such as:

```js
bcrypt.hashSync()
```

In a web server, prefer async operations.

Synchronous CPU-heavy work can block the Node.js event loop.

---

## 13. Node thread pool impact

Async bcrypt typically offloads expensive native work rather than blocking JavaScript execution directly.

But the work still consumes CPU/thread-pool resources.

High concurrent login attempts can still reduce throughput.

Rate limiting matters.

---

## 14. Thread-pool saturation

Conceptually:

```text
many bcrypt operations
      |
      v
libuv worker pool
      |
      v
queue grows
      |
      v
latency increases
```

This can affect other operations that use the same pool.

---

## 15. Login example

```js
const user =
  await User
    .findOne({
      email,
    })
    .select(
      "+passwordHash"
    );

if (!user) {
  throw new InvalidCredentialsError();
}

const matches =
  await bcrypt.compare(
    password,
    user.passwordHash
  );

if (!matches) {
  throw new InvalidCredentialsError();
}
```

Generic failure avoids account enumeration.

---

## 16. Signup example

```js
const passwordHash =
  await bcrypt.hash(
    password,
    12
  );

await User.create({
  email,
  passwordHash,
});
```

Never save the plaintext password.

---

## 17. Mongoose pre-save hook

Example:

```js
userSchema.pre(
  "save",
  async function () {
    if (
      !this.isModified(
        "password"
      )
    ) {
      return;
    }

    this.password =
      await bcrypt.hash(
        this.password,
        12
      );
  }
);
```

Important:

Without:

```js
isModified("password")
```

an existing hash may be hashed again.

---

## 18. Better naming

Prefer:

```text
passwordHash
```

over:

```text
password
```

in persistence.

This makes it harder to confuse raw password with stored hash.

---

## 19. bcrypt length limitation

Important advanced point:

bcrypt historically only processes the first **72 bytes** of password input.

This is bytes, not necessarily characters.

Unicode characters may occupy multiple bytes.

This matters when allowing very long passphrases.

---

## 20. What to do about bcrypt length behavior

Options depend on security policy:

- enforce a reasonable maximum
- use a modern alternative such as Argon2
- carefully use standardized pre-hashing designs only if you fully understand them

Do not invent your own crypto scheme casually.

---

## 21. bcrypt vs Argon2

bcrypt:
- mature
- widely supported
- battle-tested
- CPU-focused

Argon2:
- modern
- memory-hard
- stronger resistance to specialized cracking hardware
- recommended by many modern security guidelines

bcrypt remains common and acceptable in many systems when configured well.

---

## 22. bcrypt vs scrypt

scrypt is memory-hard.

bcrypt is mainly computationally expensive.

Memory-hard algorithms can make GPU/ASIC cracking more expensive.

---

## 23. Rehash detection

Stored hash contains its cost.

If policy changes from 10 to 12, after successful login:

```text
verify old hash
   |
   v
detect old cost
   |
   v
rehash with new cost
   |
   v
save new hash
```

---

## 24. Password change flow

```text
verify current password
      |
      v
validate new password
      |
      v
bcrypt.hash(newPassword)
      |
      v
save new passwordHash
      |
      v
invalidate sessions if required
```

---

## 25. Never compare bcrypt hashes directly

Wrong:

```js
const newHash =
  await bcrypt.hash(
    candidate,
    12
  );

if (
  newHash === storedHash
) {
  // ...
}
```

This fails because each hash uses a different salt.

Correct:

```js
bcrypt.compare(
  candidate,
  storedHash
)
```

---

## 26. Error handling

Do not distinguish publicly:

```text
user not found
password wrong
```

Both should usually map to:

```text
Invalid credentials
```

---

## 27. Rate limit login

Because bcrypt is intentionally expensive, login endpoints are attractive for CPU exhaustion attacks.

Use:
- IP rate limiting
- account-aware throttling
- monitoring
- MFA

---

## 28. Common mistakes

### Mistake 1
Using bcrypt synchronously in request handlers.

### Mistake 2
Comparing hashes directly.

### Mistake 3
Very low cost forever.

### Mistake 4
Extremely high cost without load testing.

### Mistake 5
Ignoring 72-byte behavior.

### Mistake 6
Rehashing an existing hash accidentally.

### Mistake 7
No login rate limit.

---

## 29. Interview questions

### Is bcrypt encryption?

No. It is a one-way password hash.

### Why do same passwords produce different hashes?

Unique random salts.

### What are salt rounds?

bcrypt cost parameter controlling computational work.

### Why use compare instead of hashing again and comparing strings?

A new hash gets a new salt, so direct hash equality is not the correct verification method.

### Why prefer async bcrypt in Node.js?

To avoid blocking the event loop with synchronous CPU-heavy work.

---

## 30. Strong interview answer

> bcrypt is a salted, adaptive password-hashing algorithm. The stored hash encodes the algorithm version, cost, salt, and derived hash, so bcrypt.compare can verify a candidate password later. I use the async API, benchmark the cost factor on production-like hardware, rate-limit authentication endpoints, and never compare independently generated bcrypt hashes because each hash uses a different random salt.

---

## Interview-Ready Summary

```text
bcrypt
   |
   +--> one-way
   +--> salted
   +--> adaptive cost
   +--> async compare

Stored string includes:
version
cost
salt
hash

Remember:
same password != same bcrypt hash
compare(), don't rehash+compare
```

## Practical Task

Build:

```text
POST /signup
POST /login
PATCH /change-password
```

Use bcrypt safely and benchmark two different cost values.
