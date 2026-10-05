# Lesson 96 — Request Validation

## Why request validation is critical

Every request coming from a client is untrusted.

That includes:

- body
- query params
- route params
- headers
- cookies

Validation protects:

- business logic
- database integrity
- security
- performance
- API consistency

---

## 1. Parsing is not validation

This JSON:

```json
{
  "email": 123,
  "age": "hello"
}
```

is syntactically valid JSON.

But it is invalid application data.

So:

```text
Parsing
   !=
Validation
```

---

## 2. Validation layers

A strong API often validates at multiple levels:

```text
Transport Validation
    |
    v
Schema Validation
    |
    v
Business Validation
    |
    v
Database Constraints
```

Each layer solves a different problem.

---

## 3. Transport validation

Examples:

- correct Content-Type
- request body size
- valid JSON
- required headers

These are protocol-level concerns.

---

## 4. Schema validation

Example requirements:

```text
email -> string + valid email
age -> integer between 18 and 100
name -> string, 2-100 chars
```

Schema tools may include:

- Zod
- Joi
- Yup
- Ajv / JSON Schema

Tool choice matters less than consistent validation.

---

## 5. Business validation

Example:

```text
quantity = 5
```

Schema says valid integer.

But business rule says:

```text
available stock = 3
```

That is not schema validation.

It belongs in business logic.

---

## 6. Database constraints

Example:

```text
email UNIQUE
```

Even if application checks first, database must enforce uniqueness.

Why?

Two concurrent requests can pass application check at the same time.

---

## 7. Validate body

Example:

```js
const schema =
  z.object({
    name:
      z.string()
        .min(2)
        .max(100),

    email:
      z.string()
        .email(),
  });
```

Then:

```js
const result =
  schema.safeParse(
    req.body
  );
```

---

## 8. Validate params

Route:

```text
/users/:id
```

Validate:
- integer
- UUID
- MongoDB ObjectId
- domain-specific ID

Do not let invalid identifiers reach your DB layer.

---

## 9. Validate query

Example:

```text
?page=1&limit=20&sort=createdAt
```

Validate:
- page >= 1
- limit <= max
- allowed sort field
- allowed order values

---

## 10. Validate headers

Example:

```text
Idempotency-Key
X-Request-Id
Content-Type
Authorization
```

Headers are also user-controlled input.

---

## 11. Whitelisting fields

Bad:

```js
await User.update(
  req.body
);
```

Attacker may send:

```json
{
  "role": "admin"
}
```

Use validated output with only allowed fields.

---

## 12. Strip unknown fields

A strong validator may either:

- reject unknown fields
- strip unknown fields

Example input:

```json
{
  "name": "Vikash",
  "role": "admin"
}
```

If role is not allowed, do not pass it forward.

---

## 13. Coercion

Query params arrive as strings.

Example:

```text
limit=20
```

Schema can safely transform:

```text
"20"
   |
   v
20
```

Be careful with implicit coercion.

Example:

```text
Boolean("false") === true
```

Explicit validation is safer.

---

## 14. Validation middleware

Example architecture:

```text
req.body
   |
   v
validateBody(schema)
   |
   v
req.validatedBody
   |
   v
controller
```

This keeps controllers clean.

---

## 15. Standard validation response

Example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request",
    "details": [
      {
        "field": "email",
        "message": "Invalid email"
      }
    ]
  }
}
```

Do not expose sensitive internal validator details unnecessarily.

---

## 16. 400 or 422?

Both conventions exist.

One common approach:

```text
400
  malformed request / invalid syntax

422
  syntactically valid but semantically invalid input
```

Pick a convention and use it consistently.

---

## 17. Validation and security

Validation helps prevent:

- injection
- oversized inputs
- unexpected types
- mass assignment
- malformed IDs
- abusive query shapes

But validation alone does not replace:
- authorization
- escaping
- parameterized SQL
- rate limiting

---

## 18. Validation and performance

Bad:

```text
GET /users?limit=10000000
```

Good validator:

```text
limit <= 100
```

Validation protects infrastructure from abusive requests.

---

## 19. Validation and normalization

Example:

```text
email:
"  VIKASH@EXAMPLE.COM "
```

Normalization may produce:

```text
vikash@example.com
```

But normalization should be deliberate and domain-aware.

---

## 20. Avoid async checks inside schema unnecessarily

Schema validation is best for structural checks.

Business checks like:

```text
email already exists?
inventory available?
coupon valid?
```

often belong in services/repositories.

Keep boundaries clear.

---

## 21. Validation race conditions

Bad flow:

```text
check email doesn't exist
   |
   v
insert user
```

Two concurrent requests can both pass the check.

Solution:

```text
application validation
+
database unique constraint
```

---

## 22. Fail fast

Reject invalid input before expensive work.

Good flow:

```text
validate
   |
   v
authenticate/authorize as appropriate
   |
   v
business logic
   |
   v
database/external APIs
```

Exact ordering depends on security and endpoint semantics.

---

## 23. Common mistakes

### Mistake 1
Validating body but ignoring params/query.

### Mistake 2
Using req.body directly in DB updates.

### Mistake 3
No field allowlist.

### Mistake 4
Treating application validation as replacement for DB constraints.

### Mistake 5
Mixing every business rule into schema validation.

### Mistake 6
No max lengths or pagination limits.

---

## 24. Interview questions

### Why validate requests?

To protect business logic, data integrity, security, and infrastructure.

### Parsing vs validation?

Parsing converts syntax into data; validation checks whether the data satisfies expected rules.

### Should DB constraints still exist if application validates?

Yes.

### Where do business rules belong?

Usually in the service/domain layer rather than basic request schema validation.

### What is mass assignment?

Passing arbitrary client fields directly into model updates, potentially allowing unauthorized fields to change.

---

## 25. Strong interview answer

> I treat every part of an incoming request as untrusted. I validate route params, query params, headers, and body before they reach business logic. Schema validation handles structure and types, while domain rules stay in the service layer and database constraints enforce invariants such as uniqueness under concurrency. I also whitelist fields to prevent mass assignment and apply limits to protect performance.

---

## Interview-Ready Summary

```text
Request
   |
   v
Transport checks
   |
   v
Schema validation
   |
   v
Normalized validated DTO
   |
   v
Business validation
   |
   v
DB constraints

Validate:
body
params
query
headers
cookies
```

## Section 9 Progress Map

```text
REST Architecture
      |
      v
Resource Design
      |
      v
Method Semantics
      |
      v
Idempotency
      |
      v
Status Codes
      |
      v
Request Validation
      |
      v
Next:
Pagination
Filtering
Sorting
Searching
Response Design
Error Design
Versioning
Rate Limiting
OpenAPI
```

## Practice Task

Create validation schemas for:

```text
POST /users
PATCH /users/:id
GET /products
POST /payments
```

Cover:
- route params
- query params
- body
- headers
- max lengths
- pagination limits
- idempotency key validation
