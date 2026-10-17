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
