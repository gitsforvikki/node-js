# Lesson 136 — Node.js API Security Fundamentals

## 1. What is API security?

An **API** lets clients (React, mobile apps, etc.) communicate with your backend. **API security** means making sure strangers cannot read, change, or overload data they should not access.

Imagine your app has `GET /api/users/42`. Being logged in does **not** mean you should be allowed to see *every* user's private data.

### Three simple questions

1. **Who are you?** Authentication: identify the user (session or token).
2. **Are you allowed?** Authorization: check access to this *specific resource*.
3. **Can we trust this input?** Validation: check the request before processing.

## 2. How an API request should flow

```text
Client request
   |
   v
HTTPS -> Rate limit -> Authentication -> Validate input
   |
   v
Authorization (can this user access THIS resource?)
   |
   v
Controller -> Service -> Database
   |
   v
Safe response (no passwords, secrets or stack traces)
```

Different applications may place these checks in different middleware stages. The important part is that **every protected action gets the right checks**.

## 3. Common risks with simple examples

| Risk | Example | Basic defense |
| --- | --- | --- |
| Broken authorization | User A requests User B's profile | Check ownership/permissions |
| Injection | User input changes a database query | Validate types + safe queries |
| Leaked secrets | API returns password hash or API key | Select safe fields and redact logs |
| Weak login protection | Attacker repeatedly guesses passwords | Rate limits + account-level controls |
| Too much information in errors | Database connection string exposed | Generic client errors; detailed server logs |
| Unprotected transport | HTTP exposes credentials in transit | Use HTTPS in production |

## 4. Express 5 practical example: check ownership

Assume earlier authentication middleware verified the session/JWT and set `req.user = { id: "..." }`. **Never trust a user ID supplied by the client as proof of identity.**

```js
// GET /api/users/:id
app.get("/api/users/:id", requireAuth, async (req, res) => {
  if (req.user.id !== req.params.id) {
    return res.status(403).json({ message: "Forbidden" });
  }

  const user = await User.findById(req.params.id)
    .select("name email")
    .lean();

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  res.json({ data: user });
});
```

**Step-by-step:** login middleware identifies requester → compare requester ID and target ID → fetch only safe fields → send response.

**Important:** This is a teaching example for a self-profile route. Real apps should use a documented permission policy, consistent ID normalization, and avoid revealing private details through error differences.

## 5. HTTP status codes for security

- **400** — malformed request, such as invalid field type.
- **401** — not authenticated or invalid authentication.
- **403** — authenticated, but not allowed to perform the action.
- **404** — resource not found (sometimes also used to avoid revealing existence).
- **429** — too many requests.
- **500** — unexpected server error; never expose internal details.

## 6. Safe error handler in Express 5

```js
app.use((err, req, res, next) => {
  console.error("Request failed:", err); // Protect server logs, too.
  res.status(500).json({ message: "Internal server error" });
});
```

This is a minimal example. In production, distinguish expected client errors from unexpected errors and redact sensitive log data. Express 5 forwards rejected Promises from async handlers to error middleware.

## 7. Practical API security checklist

- Use HTTPS and secure cookie settings in production.
- Authenticate protected routes **and** authorize each requested resource.
- Validate user input (Lesson 137).
- Use parameterized SQL (Lesson 138) and safe MongoDB filters (Lesson 139).
- Limit abusive requests and request-body sizes.
- Keep credentials in secret storage, not source code.
- Avoid sending sensitive fields, tokens, stack traces, or internal error details.
- Patch dependencies, review file uploads, and log security events safely.

## Practice

1. What is the difference between **401** and **403**?
2. In `GET /api/orders/:id`, why is checking only login status not enough?
3. Add ownership checking to an API route that returns a user's orders.
4. Explain why hiding the Delete button in React does **not** secure a backend DELETE API.

**Interview answer:** API security is layers of protection: secure transport, authentication, authorization, validation, safe database access, rate limiting, and careful error handling. The backend must enforce permissions even if the frontend hides actions.

---

## 8. Authentication vs authorization vs ownership (important)

Authentication asks **who sent the request?** Authorization asks **may they do this action?** Ownership asks **does this specific record belong to them?**

Example: Alice and Bob both have valid tokens. Alice must not be able to edit Bob's order just by changing `/orders/101` to `/orders/102`. This is **broken object-level authorization (BOLA / IDOR)**.

```js
app.patch("/orders/:id", requireAuth, async (req, res) => {
  const order = await Order.findOne({
    _id: req.params.id,
    userId: req.user.id, // Filter by owner, not just order ID
  });
  if (!order) return res.status(404).json({ message: "Not found" });
  // Validate allowed update fields before applying changes.
  res.json({ id: order.id });
});
```

A valid JWT alone is **not** permission to access every record. Also check role/function permissions (for example, ordinary users cannot call admin endpoints).

## 9. Mass assignment and excessive data exposure

**Mass assignment:** client submits fields you never intended to allow, like `role: "admin"`.

```js
// BAD
await User.findByIdAndUpdate(req.user.id, req.body);

// BETTER: validate and choose allowed fields
const parsed = z.strictObject({
  displayName: z.string().trim().min(1).max(60),
}).safeParse(req.body);
if (!parsed.success) return res.sendStatus(400);
await User.findByIdAndUpdate(req.user.id, {
  $set: { displayName: parsed.data.displayName },
});
```

**Excessive data exposure:** returning `res.json(user)` might leak `passwordHash`, recovery tokens, internal flags, or private data. Use explicit projections and response DTOs.

## 10. Security is layers, not one package

- **TLS/HTTPS:** protects data while traveling over the network.
- **Secure sessions/tokens:** identify the requester; validate signature, expiry and intended audience when applicable.
- **Authorization:** checks every sensitive action and resource.
- **Validation + safe database queries:** blocks malformed input and injection.
- **Rate limits and timeouts:** reduce abuse and resource exhaustion.
- **Logging/monitoring:** detect suspicious failures without logging passwords or tokens.
- **Updates and least privilege:** reduce the impact of vulnerabilities.

**Threat-model habit:** For each endpoint ask: Who can call it? Which data can they read/write? What if the input is huge, missing, malicious, or repeated 10,000 times?

## 11. Security test cases

Try these with your own test account:
1. Call a protected endpoint without credentials → expect 401.
2. Call another user's resource with your valid credentials → expect 403/404.
3. Add `role: "admin"` to profile update → rejected/ignored.
4. Submit a very large body → bounded request size.
5. Trigger a server error → no stack trace or secret in response.
6. Call login repeatedly → throttling and monitoring.

**Next:** Lesson 137 validates input; Lessons 138–139 secure database queries; later lessons cover XSS, CSRF, CORS and security headers.