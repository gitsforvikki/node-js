# Lesson 120 — Authentication vs Authorization

## Why this lesson matters

Authentication and authorization are among the most frequently asked backend interview topics.

A weak answer is:

> Authentication means login and authorization means permission.

That is directionally correct, but incomplete.

A strong backend engineer should understand:

- identity vs permission
- authentication factors
- sessions vs tokens
- RBAC vs ownership checks
- route-level authorization
- object-level authorization
- least privilege
- common security failures such as IDOR/BOLA
- where authentication and authorization belong in an Express architecture

---

## 1. Authentication

Authentication answers:

```text
Who are you?
```

Examples:

- username + password
- email + password
- OTP
- magic link
- biometric
- OAuth login
- API key
- client certificate

If the system successfully verifies identity, the user is authenticated.

---

## 2. Authorization

Authorization answers:

```text
What are you allowed to do?
```

Examples:

- Can this user view this order?
- Can this user delete another account?
- Can this user access the admin dashboard?
- Can this API key create invoices?

---

## 3. Core mental model

```text
Request
   |
   v
Authentication
   |
   | identify caller
   v
Authorization
   |
   | check permission
   v
Resource / Action
```

Authentication usually comes before authorization.

---

## 4. Example

Request:

```text
DELETE /users/123
```

Authentication:

```text
Token belongs to user 456
```

Authorization:

```text
Is user 456 an admin?
OR
Is user 456 allowed to delete user 123?
```

These are separate checks.

---

## 5. Authentication is not authorization

A valid token proves identity.

It does **not** automatically mean:

```text
user can access everything
```

This is a very common security mistake.

---

## 6. Authentication factors

Three common factor categories:

### Something you know

- password
- PIN

### Something you have

- phone
- hardware key
- authenticator device

### Something you are

- fingerprint
- face

Multi-factor authentication combines different factor categories.

---

## 7. Session-based authentication

Flow:

```text
Login
  |
  v
Server verifies credentials
  |
  v
Server creates session
  |
  v
Session ID stored in cookie
  |
  v
Future requests send session ID
```

Server uses session store to identify user.

---

## 8. Token-based authentication

Flow:

```text
Login
  |
  v
Server verifies credentials
  |
  v
Issues token
  |
  v
Client sends token later
  |
  v
Server verifies token
```

JWT is one token format.

Token-based does not automatically mean "stateless" in every real system, especially when revocation/refresh state is stored.

---

## 9. Cookie is not an authentication strategy

Important interview point:

```text
Cookie
  -> transport/storage mechanism in browser

JWT
  -> token format

Session
  -> server-side authentication state model
```

You can store:
- session ID in cookie
- JWT in cookie

These concepts are different.

---

## 10. Authentication middleware

Express example:

```js
async function authenticate(
  req,
  res,
  next
) {
  try {
    const token =
      extractAccessToken(req);

    if (!token) {
      return res
        .status(401)
        .json({
          error: {
            code:
              "AUTH_REQUIRED",
          },
        });
    }

    const payload =
      verifyAccessToken(token);

    const user =
      await userRepository
        .findById(
          payload.sub
        );

    if (!user) {
      return res
        .status(401)
        .json({
          error: {
            code:
              "INVALID_SESSION",
          },
        });
    }

    req.user = user;

    next();
  } catch (error) {
    next(error);
  }
}
```

---

## 11. Authorization middleware

Role example:

```js
function requireRole(
  ...roles
) {
  return (
    req,
    res,
    next
  ) => {
    if (
      !roles.includes(
        req.user.role
      )
    ) {
      return res
        .status(403)
        .json({
          error: {
            code:
              "FORBIDDEN",
          },
        });
    }

    next();
  };
}
```

---

## 12. RBAC

RBAC means:

```text
Role-Based Access Control
```

Example:

```text
USER
ADMIN
SUPPORT
MANAGER
```

Permissions are assigned to roles.

Example:

```text
ADMIN
  -> delete user
  -> manage products
  -> view reports
```

---

## 13. RBAC limitations

Role checks alone are often not enough.

Suppose:

```text
role = user
```

A user may view:

```text
their own order
```

but not:

```text
another user's order
```

This requires resource-level authorization.

---

## 14. Ownership authorization

Bad:

```js
const order =
  await Order.findById(
    req.params.id
  );
```

Then return it to any authenticated user.

This creates an object-level authorization vulnerability.

Better:

```js
const order =
  await Order.findOne({
    _id:
      req.params.id,
    userId:
      req.user.id,
  });
```

The authorization constraint is part of the query.

---

## 15. BOLA / IDOR

Common terms:

```text
IDOR
  -> Insecure Direct Object Reference

BOLA
  -> Broken Object Level Authorization
```

Example attack:

```text
GET /orders/1001
GET /orders/1002
GET /orders/1003
```

If changing the ID reveals other users' orders, authorization is broken.

---

## 16. Function-level authorization

Example:

```text
DELETE /admin/users/123
```

Even if object ownership is irrelevant, only privileged roles should access the function.

This is function-level authorization.

---

## 17. Object-level authorization

Example:

```text
GET /orders/:id
```

Check whether caller can access **that specific order**.

---

## 18. Field-level authorization

Sometimes users can access a resource but not every field.

Example:

Support agent may see:

```text
name
email
order status
```

but not:

```text
password hash
private security flags
payment secrets
```

Authorization can exist at field level too.

---

## 19. Policy-based authorization

Instead of only:

```text
role === admin
```

you can use policies:

```text
canEditOrder(user, order)
```

Example:

```js
function canCancelOrder(
  user,
  order
) {
  return (
    user.role === "admin" ||
    (
      order.userId ===
        user.id &&
      order.status ===
        "pending"
    )
  );
}
```

This can be clearer for complex business rules.

---

## 20. ABAC

ABAC:

```text
Attribute-Based Access Control
```

Decision may depend on:

- user role
- department
- resource owner
- location
- time
- subscription tier
- resource status

More flexible than pure RBAC.

---

## 21. Principle of least privilege

Give users/services only the permissions they need.

Bad:

```text
every authenticated user -> admin-like access
```

Good:

```text
minimum required permission
```

This reduces blast radius.

---

## 22. Deny by default

A strong authorization model should conceptually be:

```text
not explicitly allowed
      |
      v
deny
```

rather than:

```text
not explicitly denied
      |
      v
allow
```

---

## 23. Authentication state freshness

Suppose JWT says:

```text
role = admin
```

but user was demoted five minutes later.

If token is long-lived, stale authorization data may remain valid until expiration.

Possible strategies:

- short access token TTL
- DB lookup for sensitive actions
- token version
- revocation state
- session-based authorization

---

## 24. 401 vs 403

```text
401 Unauthorized
  -> actually means unauthenticated / invalid credentials

403 Forbidden
  -> authenticated but not allowed
```

Mental model:

```text
Identity known?
   |
   +--> no -> 401
   |
   +--> yes
          |
          v
Allowed?
   |
   +--> no -> 403
```

---

## 25. Sometimes use 404 instead of 403

For sensitive resources:

```text
GET /private-files/123
```

If caller should not know resource exists, returning 404 may reduce information leakage.

This is a deliberate security choice.

---

## 26. Authentication flow example

```text
POST /login
   |
   v
validate input
   |
   v
find user
   |
   v
verify password hash
   |
   v
issue session/token
   |
   v
client stores credential safely
```

---

## 27. Authorization flow example

```text
GET /orders/123
   |
   v
verify identity
   |
   v
load order with ownership constraint
   |
   v
allowed?
   |
   +--> yes -> return
   |
   +--> no -> deny
```

---

## 28. Common mistakes

### Mistake 1
Treating authentication as authorization.

### Mistake 2
Only checking role, never resource ownership.

### Mistake 3
Trusting userId from request body instead of authenticated identity.

### Mistake 4
Long-lived tokens with stale permission claims.

### Mistake 5
Returning sensitive fields just because user is authenticated.

### Mistake 6
Default allow instead of default deny.

---

## 29. Interview questions

### Authentication vs authorization?

Authentication verifies identity. Authorization determines what that identity is allowed to do.

### What is RBAC?

Permissions are assigned to roles, and users receive permissions through roles.

### What is BOLA/IDOR?

A vulnerability where users can access objects they are not authorized to access simply by changing identifiers.

### Why is a valid JWT not enough?

It proves claims/identity, but resource-specific permission still needs authorization.

### 401 vs 403?

401 for missing/invalid authentication; 403 for authenticated but forbidden access.

---

## 30. Strong interview answer

> Authentication establishes who the caller is, while authorization decides whether that caller may perform a specific action on a specific resource. In Express I usually authenticate first, attach a trusted identity to req.user, and then enforce authorization through role checks, ownership constraints, or policy functions. I never trust a resource ID alone, because that leads to BOLA/IDOR vulnerabilities, and I prefer deny-by-default and least-privilege access.

---

## Interview-Ready Summary

```text
Authentication
  -> Who are you?

Authorization
  -> What can you do?

Authorization types:
RBAC
ownership
policy-based
ABAC
field-level

Security:
least privilege
deny by default
prevent BOLA/IDOR
```

## Practical Task

Implement:

```text
GET /orders/:id
DELETE /admin/users/:id
PATCH /profile
```

Requirements:

- authenticate every protected route
- enforce order ownership
- enforce admin role
- never trust userId from body
