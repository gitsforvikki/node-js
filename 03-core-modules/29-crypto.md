# Lesson 29 — crypto

## Core Concept

The `crypto` module provides cryptographic primitives.

Import:

```js
import crypto from "node:crypto";
```

It supports:
- secure random values
- hashing
- HMAC
- encryption primitives
- signatures
- key generation

Cryptography is easy to misuse, so prefer established protocols and libraries instead of inventing your own schemes.

## Random Bytes

```js
const token = crypto
  .randomBytes(32)
  .toString("hex");

console.log(token);
```

Useful for:
- reset tokens
- API secrets
- nonces
- session identifiers

Do not use `Math.random()` for security-sensitive tokens.

## randomUUID()

```js
const id = crypto.randomUUID();

console.log(id);
```

Useful for unique identifiers.

## Hashing

```js
const hash = crypto
  .createHash("sha256")
  .update("hello")
  .digest("hex");
```

Hash properties:
- one-way transformation
- deterministic
- fixed-size output

## Hashing Is Not Encryption

```text
Hash
  data -> digest
  no normal reverse operation

Encryption
  plaintext -> ciphertext
  key can decrypt
```

## HMAC

HMAC combines a secret key with a hash function.

```js
const signature = crypto
  .createHmac("sha256", secret)
  .update(payload)
  .digest("hex");
```

Common use:
- webhook verification
- message integrity
- signed requests

## Webhook Verification Example

```js
const expected = crypto
  .createHmac("sha256", webhookSecret)
  .update(rawBody)
  .digest("hex");
```

Then compare against the provider's signature.

## Timing-Safe Comparison

Avoid naive secret comparisons in security-sensitive contexts.

Node provides:

```js
crypto.timingSafeEqual(a, b);
```

Buffers must be the same length.

Example:

```js
const a = Buffer.from(expected);
const b = Buffer.from(received);

const valid =
  a.length === b.length &&
  crypto.timingSafeEqual(a, b);
```

## Password Hashing

Do not use a fast general-purpose hash like plain SHA-256 for user passwords.

Bad:

```js
sha256(password)
```

Passwords need intentionally slow password hashing algorithms such as:
- bcrypt
- scrypt
- Argon2

Node provides `scrypt` support.

## scrypt

Conceptual async usage:

```js
crypto.scrypt(
  password,
  salt,
  64,
  (err, derivedKey) => {
    // store derived key + salt
  }
);
```

In production, define parameters carefully and follow current security guidance.

## Encryption

The crypto module supports symmetric encryption primitives such as AES.

But secure encryption requires correct management of:
- algorithm
- key
- IV/nonce
- authentication tag
- key rotation

Use authenticated encryption modes and proven designs.

## Common Mistakes

- using Math.random for tokens
- hashing passwords with SHA-256
- inventing custom crypto
- comparing signatures with normal string equality
- reusing IVs/nonces incorrectly
- hardcoding secrets
- converting/normalizing webhook payload before signature verification when provider requires raw bytes

## Industry Example

Payment webhook:

```text
Provider
   |
   | payload + signature
   v
Node API
   |
   +--> calculate HMAC with secret
   |
   +--> timing-safe compare
   |
   +--> process event only if valid
```

## Interview Questions

### Hashing vs encryption?

Hashing is one-way. Encryption is reversible with a key.

### What is HMAC?

A keyed message authentication mechanism used to verify message integrity and authenticity.

### Why is SHA-256 not suitable by itself for passwords?

It is intentionally fast, making brute-force attacks cheaper. Passwords require slow, password-specific derivation functions.

## Summary

```text
crypto
  randomBytes/randomUUID
  hashing
  HMAC
  timingSafeEqual
  password KDFs
  encryption/signatures
```

Security rule:

> Use primitives correctly, and prefer established designs over custom cryptographic protocols.

## Practice Task

Build a webhook verification helper that:
- accepts raw payload
- computes HMAC SHA-256
- compares signatures safely
- rejects invalid signatures
