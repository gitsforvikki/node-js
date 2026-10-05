# Lesson 131 — Authentication Middleware

## Why authentication middleware matters

Authentication logic should not be duplicated in every route.

Bad:

```js
router.get(
  "/profile",
  async (req, res) => {
    // manually read token
    // verify token
    // load user
  }
);
```

Then repeat the same logic for every protected endpoint.

Authentication middleware centralizes identity verification.

---

# 1. Middleware Responsibility

Authentication middleware should answer:

```text
Is this request authenticated?
Who is the caller?
```

It should not contain unrelated business logic.

---

# 2. Pipeline

```text
Request
  |
  v
Authentication Middleware
  |
  +--> invalid -> 401
  |
  +--> valid
          |
          v
      req.user / req.auth
          |
          v
      Authorization
          |
          v
      Controller
```

---

# 3. Authentication vs Authorization Middleware

Authentication:

```text
Who are you?
```

Authorization:

```text
Are you allowed?
```

Keep them separate.

---

# 4. JWT Middleware Flow

```text
Authorization header
      |
      v
Extract Bearer token
      |
      v
Verify signature
      |
      v
Validate exp/iss/aud
      |
      v
Build trusted identity
      |
      v
req.auth
```

---

# 5. Example JWT Middleware

```js
async function authenticate(
  req,
  res,
  next
) {
  try {
    const header =
      req.headers.authorization;

    if (!header) {
      return res
        .status(401)
        .json({
          error: {
            code:
              "AUTH_REQUIRED",
          },
        });
    }

    const [
      scheme,
      token,
    ] =
      header.split(" ");

    if (
      scheme !== "Bearer" ||
      !token
    ) {
      return res
        .status(401)
        .json({
          error: {
            code:
              "INVALID_AUTH_SCHEME",
          },
        });
    }

    const payload =
      await verifyAccessToken(
        token
      );

    req.auth = {
      userId:
        payload.sub,
      role:
        payload.role,
    };

    next();
  } catch (error) {
    next(error);
  }
}
```

---

# 6. Do Not Decode Only

Wrong:

```js
const payload =
  decode(token);

req.userId =
  payload.sub;
```

An attacker can modify decoded claims.

Correct:

```text
verify
then trust
```

---

# 7. Session Middleware Flow

```text
Cookie
  |
  v
session ID
  |
  v
Session Store
  |
  v
session data
  |
  v
req.user
```

---

# 8. Example Session Authentication

```js
async function requireSession(
  req,
  res,
  next
) {
  if (
    !req.session?.userId
  ) {
    return res
      .status(401)
      .json({
        error: {
          code:
            "AUTH_REQUIRED",
        },
      });
  }

  req.auth = {
    userId:
      req.session.userId,
  };

  next();
}
```

---

# 9. Should Middleware Load User From DB?

Two valid patterns exist.

### Pattern A — Token only

```text
verify JWT
   |
   v
req.auth = claims
```

Advantages:
- fast
- no DB lookup

Trade-off:
- stale user status/role possible

---

### Pattern B — Token + DB lookup

```text
verify JWT
   |
   v
load user
   |
   v
check active status
   |
   v
req.user
```

Advantages:
- fresh state
- disabled users blocked immediately

Trade-off:
- DB/cache lookup per request

---

# 10. Hybrid Strategy

Common:

Normal route:

```text
trust short-lived verified token
```

Sensitive route:

```text
verify token
+
load fresh user/permissions
```

Good balance.

---

# 11. req.auth vs req.user

Useful convention:

```text
req.auth
  -> trusted credential claims

req.user
  -> loaded database user
```

Example:

```js
req.auth = {
  userId: "usr_123",
  role: "admin",
};

req.user = {
  id: "usr_123",
  email: "...",
  active: true,
};
```

This keeps concepts clear.

---

# 12. Missing Credential

Return:

```text
401
```

Example code:

```text
AUTH_REQUIRED
```

---

# 13. Expired Token

Usually:

```text
401
```

Application code:

```text
ACCESS_TOKEN_EXPIRED
```

Client may trigger refresh flow.

---

# 14. Invalid Signature

Return generic authentication failure.

Do not reveal crypto internals.

---

# 15. Wrong Audience / Issuer

Treat token as invalid.

Do not accept a validly signed token intended for another service.

---

# 16. Disabled User

If middleware loads user:

```text
user.active = false
```

deny access.

Often:

```text
401 or account-specific 403
```

depending on your API semantics.

---

# 17. Deleted User

If token identifies a user that no longer exists:

```text
authentication should fail
```

Do not let orphan tokens continue privileged access.

---

# 18. Token Version

Middleware can compare:

```text
token.ver
vs
user.tokenVersion
```

Mismatch:

```text
reject token
```

Useful after:
- password reset
- logout all devices
- security incident

---

# 19. Authorization Comes After Authentication

Example:

```js
router.delete(
  "/users/:id",
  authenticate,
  requirePermission(
    "users.delete"
  ),
  deleteUser
);
```

Clear separation.

---

# 20. Optional Authentication

Some routes work for both anonymous and authenticated users.

Example:

```text
GET /products
```

Authenticated user may receive personalized result.

Optional auth middleware:

```text
credential absent
  -> continue anonymously

credential present and valid
  -> attach identity

credential malformed
  -> maybe reject
```

Define semantics clearly.

---

# 21. Middleware Factory

Different auth strategies:

```js
function authenticateWith(
  verifier
) {
  return async (
    req,
    res,
    next
  ) => {
    // ...
  };
}
```

Useful for reusable architecture.

---

# 22. API Key Middleware

Not every authentication middleware uses JWT.

Example:

```text
X-API-Key
   |
   v
hash/look up key
   |
   v
client identity
```

Useful for machine clients.

---

# 23. Service Token Middleware

Internal services may authenticate using:

- signed JWT
- mTLS
- API key
- workload identity

Do not assume user authentication model should be reused unchanged.

---

# 24. Cookie Authentication Middleware

Example:

```js
const token =
  req.cookies.accessToken;
```

If using cookies:
- SameSite
- CSRF
- Secure
- HttpOnly

must be considered.

---

# 25. Header Authentication Middleware

Example:

```text
Authorization: Bearer token
```

Common for APIs and mobile apps.

---

# 26. Error Mapping

Crypto library may throw:

```text
TokenExpiredError
JsonWebTokenError
```

Do not expose those raw class names.

Map:

```text
TokenExpiredError
  -> ACCESS_TOKEN_EXPIRED

invalid signature
  -> INVALID_TOKEN
```

---

# 27. Centralized Error Handler

Authentication middleware can pass typed errors:

```js
throw new AuthError(
  "ACCESS_TOKEN_EXPIRED"
);
```

Central error middleware converts to response.

This keeps response format consistent.

---

# 28. Avoid Duplicate Responses

Bad:

```js
res.status(401).json(...);
next();
```

After sending response, return.

Correct:

```js
return res
  .status(401)
  .json(...);
```

---

# 29. Never Trust userId From Body

Bad:

```js
const userId =
  req.body.userId;
```

for identity.

Use:

```js
req.auth.userId
```

Trusted identity comes from verified credential.

---

# 30. Ownership Query

Instead of:

```js
Order.findById(
  req.params.id
);
```

for normal user:

```js
Order.findOne({
  _id:
    req.params.id,
  userId:
    req.auth.userId,
});
```

Authentication middleware enables secure downstream ownership checks.

---

# 31. Sensitive Endpoint Reauthentication

Example:

```text
DELETE /account
```

May require:

- fresh token
- password confirmation
- MFA

Middleware can enforce:

```text
auth_time < 5 minutes ago
```

or equivalent.

---

# 32. Authentication Middleware Ordering

Good:

```text
request ID
  |
  v
logging
  |
  v
body parser
  |
  v
authentication
  |
  v
authorization
  |
  v
validation/business route
```

Exact ordering can vary by route.

---

# 33. Validation Before or After Authentication?

Depends on what is being validated.

Security principle:

Do not perform expensive work for unauthenticated callers unnecessarily.

Typical protected route:

```text
authenticate
   |
   v
authorize
   |
   v
validate request
```

But basic protocol/body parsing must happen earlier.

---

# 34. Logging Authentication Failures

Useful:

```text
requestId
IP
route
failure code
userId if trusted
timestamp
```

Never log:
- full JWT
- refresh token
- session ID
- password

---

# 35. Rate Limiting

Authentication endpoints and middleware-related failure paths should be rate-limited.

Examples:

- login
- refresh
- password reset
- API key authentication

This prevents brute force and CPU/resource abuse.

---

# 36. Caching User Lookup

If middleware loads user every request:

```text
token -> user ID
     |
     v
Redis cache
     |
     +--> hit
     |
     +--> miss -> DB
```

But cached authorization data needs invalidation.

---

# 37. TypeScript Request Extension

Concept:

```ts
declare global {
  namespace Express {
    interface Request {
      auth?: {
        userId: string;
        role?: string;
      };
    }
  }
}
```

This makes middleware-added fields type-safe.

---

# 38. Testing Authentication Middleware

Test:

- no token
- malformed token
- expired token
- wrong signature
- wrong audience
- valid token
- disabled user
- token-version mismatch
- session missing
- revoked session

---

# 39. Unit vs Integration Testing

Unit test:
- verifier mocked
- middleware behavior

Integration test:
- real signed token
- actual route
- real auth config

Both are useful.

---

# 40. Common Mistakes

### Mistake 1
Decode without verify.

### Mistake 2
Authentication and authorization mixed together.

### Mistake 3
Trusting userId from body.

### Mistake 4
Logging tokens.

### Mistake 5
No issuer/audience validation.

### Mistake 6
Middleware loads full user unnecessarily for every route.

### Mistake 7
No disabled-user strategy.

### Mistake 8
No tests for expired/revoked credentials.

---

# 41. Interview Questions

## What does authentication middleware do?

It extracts and verifies a credential, establishes trusted caller identity, and attaches that identity to the request.

## Should it perform authorization?

Prefer separate authorization middleware/policies.

## Should it query DB on every request?

Not always. It depends on freshness and security requirements.

## What should be attached to req?

Minimal trusted identity such as userId, role/scope, and optionally loaded user.

## What status for missing token?

401.

---

# 42. Strong Interview Answer

> Authentication middleware centralizes credential verification. It extracts a session ID, bearer token, API key, or other credential, verifies it cryptographically or against a trusted store, validates expiry and context such as issuer/audience, and attaches a trusted identity to the request. I keep authorization separate, never trust identity fields from request data, avoid logging credentials, and selectively load fresh user state when immediate revocation or permission freshness is required.

---

# Interview-Ready Summary

```text
Credential
   |
   v
Extract
   |
   v
Verify
   |
   v
Validate claims/state
   |
   v
req.auth
   |
   v
Authorization
   |
   v
Controller

Auth middleware should:
identify
verify
attach trusted context

It should NOT:
run business workflows
trust request userId
leak token details
```

## Section 11 Progress Map

```text
Authentication vs Authorization
        |
        v
Password Hashing
        |
        v
bcrypt
        |
        v
JWT
        |
        v
Access / Refresh Tokens
        |
        v
Cookies
        |
        v
Session Authentication
        |
        v
JWT vs Sessions
        |
        v
RBAC
        |
        v
Authentication Middleware
        |
        v
Next:
OAuth
Password Reset
Email Verification
Token Rotation / Revocation
```

## Practical Task

Create authentication middleware for:

1. JWT bearer access token
2. session cookie
3. API key

Then reuse a common authorization middleware:

```text
requirePermission("orders.read")
```
