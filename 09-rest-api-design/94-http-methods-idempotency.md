# Lesson 94 — HTTP Methods and Idempotency

## Why this lesson matters

Idempotency is one of the most frequently misunderstood API interview topics.

It matters because real distributed systems retry requests.

Retries happen due to:

- timeouts
- network failures
- proxy failures
- client crashes
- duplicated messages

Without idempotency, retries can create duplicate side effects.

---

## 1. What is idempotency?

An operation is idempotent if performing the same operation multiple times has the same intended effect as performing it once.

Example:

```text
DELETE /users/123
```

First request:
- user deleted

Second request:
- user already gone

Final state:

```text
user does not exist
```

The operation is idempotent even if responses differ.

---

## 2. Response equality is not required

This is a critical interview point.

First DELETE might return:

```text
204
```

Second might return:

```text
404
```

The method can still be idempotent because the intended resource state is unchanged.

---

## 3. GET

GET is expected to be:

```text
safe
idempotent
```

Example:

```text
GET /users/123
```

Repeated calls should not intentionally mutate the resource.

---

## 4. POST

POST is generally **not assumed idempotent**.

Example:

```text
POST /orders
```

Repeat twice:

```text
Order A created
Order B created
```

That is why payment/order APIs often add idempotency keys.

---

## 5. PUT

PUT is semantically idempotent.

Example:

```http
PUT /users/123

{
  "name": "Vikash"
}
```

Repeat it 10 times.

Final intended state remains:

```text
name = Vikash
```

---

## 6. PATCH

PATCH may or may not be idempotent.

Example idempotent PATCH:

```json
{
  "status": "active"
}
```

Repeat:
- same final status

Non-idempotent PATCH:

```json
{
  "incrementBalanceBy": 100
}
```

Repeat:
- balance changes repeatedly

So PATCH idempotency depends on semantics.

---

## 7. DELETE

DELETE is semantically idempotent.

Repeated deletion leaves the resource absent.

---

## 8. Safe vs idempotent

Safe:

> intended not to modify server resource state.

Idempotent:

> repeated execution has the same intended effect.

Examples:

```text
GET
  safe + idempotent

PUT
  not safe + idempotent

DELETE
  not safe + idempotent

POST
  not safe + generally non-idempotent
```

---

## 9. Why idempotency matters in payments

Scenario:

```text
Client
   |
   v
POST /payments
   |
   v
Payment succeeds
   |
   v
Network response lost
```

Client thinks request failed.

It retries.

Without protection:

```text
Payment 1
Payment 2
```

This is disastrous.

---

## 10. Idempotency key

Client sends:

```http
Idempotency-Key: 123e4567
```

Server stores:

```text
key
request fingerprint
result
status
```

Retry with same key:

```text
return previous result
do not execute side effect twice
```

---

## 11. Idempotency storage

Conceptual table:

```text
idempotency_keys

key
user_id
request_hash
response_status
response_body
created_at
expires_at
```

---

## 12. Request fingerprint

If the same key is reused with a different payload, that should usually be rejected.

Example:

First request:

```json
{
  "amount": 1000
}
```

Second request same key:

```json
{
  "amount": 5000
}
```

That is dangerous.

Store/compare a request hash.

---

## 13. Concurrency race

Two identical requests can arrive at the same time.

Bad flow:

```text
Request A checks key -> absent
Request B checks key -> absent
A creates payment
B creates payment
```

You need atomic protection.

Possible mechanisms:

- unique DB constraint
- transactional insert
- distributed lock
- provider idempotency support

---

## 14. In-progress requests

What if a second request arrives while the first is still processing?

Possible strategy:

```text
key state = PROCESSING
```

Then:
- wait
- return 409/425-style application behavior
- poll later
- replay once complete

Exact contract is API-specific.

---

## 15. Idempotency and retries

Safe retry strategy:

```text
transient failure
   |
   v
retry with same idempotency key
```

This allows retry without duplicating the operation.

---

## 16. Idempotency does not mean transaction

Even with idempotency, multi-step workflows may need:
- transactions
- sagas
- compensating actions

Idempotency prevents duplicate logical operation execution.

It does not make every internal step atomic.

---

## 17. Idempotency vs deduplication

Related but different.

Deduplication:

```text
detect duplicate request/message
```

Idempotency:

```text
repeated operation produces same intended effect
```

Deduplication is one technique for achieving idempotency.

---

## 18. External provider example

Payment providers often support their own idempotency keys.

Your backend may also need its own idempotency layer because:
- DB writes
- order creation
- provider calls
- webhook processing

can all be retried independently.

---

## 19. Webhook idempotency

Webhooks are often delivered more than once.

Store provider event ID:

```text
event_abc123
```

Before processing:

```text
already processed?
   |
   +--> yes -> ignore/replay safe response
   |
   +--> no -> process and record event
```

---

## 20. Common mistakes

### Mistake 1
Thinking idempotency means same HTTP response every time.

Wrong.

### Mistake 2
Thinking PATCH is always idempotent.

Wrong.

### Mistake 3
Using idempotency key without atomic uniqueness.

Race condition.

### Mistake 4
Allowing same key with different payloads.

Dangerous.

### Mistake 5
Ignoring webhook duplicate delivery.

---

## 21. Interview questions

### What is idempotency?

Repeating the same logical operation has the same intended effect as executing it once.

### Is POST idempotent?

Not by default.

### Is DELETE idempotent?

Yes, semantically.

### Is PATCH idempotent?

Depends on operation semantics.

### How do you make payment creation idempotent?

Use an idempotency key, persistent atomic storage, request fingerprinting, and return/replay the original result for retries.

---

## 22. Strong interview answer

> Idempotency means repeated execution of the same logical operation has the same intended effect as a single execution. GET, PUT, and DELETE are semantically idempotent, while POST generally is not and PATCH depends on the update semantics. For critical POST operations such as payments, I use an idempotency key stored atomically with the request fingerprint and result so retries cannot create duplicate side effects.

---

## Interview-Ready Summary

```text
GET
  safe + idempotent

POST
  usually non-idempotent

PUT
  idempotent

PATCH
  depends

DELETE
  idempotent

Critical writes:
Idempotency-Key
+ atomic persistence
+ request hash
+ replay result
```

## Practice Task

Design an idempotent:

```text
POST /payments
```

Handle:
- first request
- retry after timeout
- duplicate concurrent request
- same key with different payload
- expired idempotency key
