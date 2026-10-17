# Lesson 150 — OWASP API Security Concepts

## 1. What is OWASP?

**OWASP** stands for Open Worldwide Application Security Project. It publishes free security guidance for developers.

The **OWASP API Security Top 10 (2023)** is a widely used list of important API security risk categories. It is **not a complete checklist or automatic certification**. Use it to understand common design and implementation mistakes.

Official reference: https://owasp.org/API-Security/editions/2023/en/0x11-t10/

## 2. Why do API risks matter?

A React/Next.js frontend may hide an Admin button, but attackers can send HTTP requests directly. API security must be enforced on the **backend**.

Example: a logged-in user changes `/api/orders/123` to `/api/orders/124`. If the API returns someone else's order, it has a **broken object-level authorization** problem.

## 3. OWASP API Top 10 (2023) in simple language

| ID | Risk | Meaning | Example defense |
| --- | --- | --- | --- |
| API1 | Broken Object Level Authorization (BOLA) | User accesses another user's record | Filter by owner/tenant |
| API2 | Broken Authentication | Login/session identity can be bypassed or abused | Strong auth, MFA, rate limits |
| API3 | Broken Object Property Level Authorization | User reads/writes fields they should not | Allowlist response/update fields |
| API4 | Unrestricted Resource Consumption | Requests exhaust CPU, memory, DB, or paid APIs | Limits, quotas, timeouts |
| API5 | Broken Function Level Authorization | Ordinary user calls admin-only action | Role/function permission checks |
| API6 | Unrestricted Access to Sensitive Business Flows | Automation abuses business operations | Business-specific abuse controls |
| API7 | Server Side Request Forgery (SSRF) | Server fetches attacker-chosen internal URL | Restrict outbound destinations |
| API8 | Security Misconfiguration | Unsafe defaults or exposed debug features | Harden deployment and headers |
| API9 | Improper Inventory Management | Forgotten/old APIs remain exposed | Maintain API inventory and retire old versions |
| API10 | Unsafe Consumption of APIs | Blindly trusting third-party API responses | Validate external data and limit trust |

## 4. API1: Broken Object Level Authorization (BOLA)

**Unsafe:**

```js
app.get("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  res.json(order);
});
```

**Safer:**

```js
app.get("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findOne({
    _id: req.params.id,
    userId: req.user.id,
  }).select("items total status");
  if (!order) return res.sendStatus(404);
  res.json(order);
});
```

**Flow:** authenticated user → target order ID → database query scoped to user → only their order can be returned. Validate the ID to avoid a database casting error.

## 5. API2: Broken Authentication

Risks include weak password storage, accepting invalid tokens, no login rate limits, and insecure session handling.

Defenses: password-specific hashing, verified JWT/session, expiration and revocation, secure cookies, MFA/passkeys, generic login errors, and rate limits. See Lessons 145 and 149.

## 6. API3: Broken Object Property Level Authorization

Two related problems:

**Excessive property exposure:**

```js
// BAD: may expose passwordHash or private flags
res.json(user);

// BETTER: explicit public response
res.json({ id: user.id, name: user.name });
```

**Mass assignment:**

```js
// BAD: attacker might submit { role: "admin" }
await User.findByIdAndUpdate(req.user.id, req.body);

// BETTER: validate and update allowed fields only
await User.updateOne(
  { _id: req.user.id },
  { $set: { displayName: validated.displayName } }
);
```

## 7. API4: Unrestricted Resource Consumption

An attacker may send giant request bodies, expensive queries, or thousands of requests.

```js
app.use(express.json({ limit: "16kb" }));

const users = await User.find({ active: true })
  .limit(50)
  .maxTimeMS(2000);
```

Combine body-size limits, pagination, query timeouts, rate limits, concurrency controls, and monitoring. Limits depend on endpoint needs.

## 8. API5: Broken Function Level Authorization

A user may be allowed to view their profile but not delete another user's account or access admin features.

```js
function requireAdmin(req, res, next) {
  if (req.user?.role !== "admin") return res.sendStatus(403);
  next();
}

app.delete("/admin/users/:id", requireAuth, requireAdmin, deleteUser);
```

**Important:** `req.user.role` must come from a **trusted, verified source**. Hiding the admin button in React is not protection.

## 9. API6: Sensitive Business Flow Abuse

Even properly authenticated users can abuse valuable business operations: claiming coupons repeatedly, reserving inventory in bulk, or placing duplicate orders.

Example checkout defense:
- Enforce business limits (e.g., coupon once per eligible account).
- Use idempotency keys to avoid accidental duplicate charges.
- Validate price and stock **on the server**.
- Monitor suspicious velocity and automate abuse controls.

**Important:** A simple IP limit may not stop many accounts abusing the same business process.

## 10. API7: SSRF — Server-Side Request Forgery

SSRF happens when a server fetches a URL chosen by a client without proper restrictions.

```js
// DANGEROUS when req.body.url is arbitrary
const response = await fetch(req.body.url);
```

Attackers may try to reach internal services or cloud metadata endpoints.

**Safer design:** Avoid arbitrary URL fetching. If required, allowlist specific HTTPS hosts, reject internal/private/link-local addresses, verify DNS and redirect destinations, enforce timeouts and response-size limits, and restrict outbound network access. Simple string-prefix checks are not enough.

## 11. API8: Security Misconfiguration

Examples: debug stack traces in production, broad CORS with credentials, default admin accounts, public storage buckets, overly permissive IAM, and outdated components.

Fix by reviewing deployment configuration, applying least privilege, using Helmet/security headers where appropriate, restricting CORS, and testing error responses.

## 12. API9: Improper Inventory Management

Teams may forget old endpoints such as `/api/v1/admin` while shipping `/api/v2`. Attackers can target the unmaintained version.

Maintain an inventory of routes, owners, versions, environments, data types, and retirement dates. Remove obsolete APIs and avoid exposing development/test systems publicly.

## 13. API10: Unsafe Consumption of APIs

Data from a third-party service is **untrusted input**, even if it comes from a well-known provider.

Example: a payment webhook must have its provider signature verified using the **raw request body** as specified by that provider. Don't trust a client-supplied `paymentStatus: "paid"` or a third-party response without validation.

Also use timeouts, response-size limits, schema validation, safe logging, and correct error handling for outbound integrations.

## 14. End-to-end example: secure order API checklist

For `PATCH /api/orders/:id`:
1. Verify login/session.
2. Validate order ID and update fields.
3. Check user owns the order or has a permitted role.
4. Enforce allowed status transitions (e.g., cannot change DELIVERED back to CREATED).
5. Apply database update atomically where needed.
6. Rate limit and log meaningful security events.
7. Return only permitted fields.

```text
Request -> auth -> validation -> authorization
        -> business rules -> safe database update
        -> limited response
```

## 15. OWASP Top 10 vs other security topics

SQL/NoSQL injection, XSS, CSRF, secure uploads, and dependency vulnerabilities still matter, even if they are not all named as standalone entries in the **API Top 10 2023**. Different OWASP lists have different scopes (for example, the OWASP Web Application Top 10).

**Never assume that implementing exactly ten controls makes an API secure.**

## 16. Practice and interview questions

1. Explain BOLA vs broken function-level authorization.
2. Why can `User.findByIdAndUpdate(id, req.body)` be dangerous?
3. How can a rate-limited API still suffer business-flow abuse?
4. Why is fetching a user-provided URL a potential SSRF issue?
5. What should you verify before trusting a payment webhook?
6. Design security checks for a `POST /checkout` endpoint.

**Interview answer:** The OWASP API Security Top 10 (2023) highlights risks including broken authorization, authentication, resource abuse, SSRF, misconfiguration, poor API inventory, and unsafe third-party integrations. Secure APIs need checks at the object, property, function, and business-flow levels—not just login.
