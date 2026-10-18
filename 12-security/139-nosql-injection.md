# Lesson 139 — NoSQL Injection

## 1. What is NoSQL injection?

NoSQL injection happens when untrusted input changes the **meaning of a NoSQL database query**.

MongoDB queries are JavaScript-like objects. Some keys, such as `$ne`, `$gt`, and `$or`, are **operators** rather than ordinary field names. Allowing a user to send arbitrary query objects is dangerous.

**Remember:** SQL injection targets SQL query text; MongoDB operator injection commonly abuses unexpected objects and operators.

## 2. Unsafe login lookup example

Let's understand **NoSQL operator injection** using an Express + Mongoose example. The following code is intentionally unsafe and is **not a complete login implementation**.

### 2.1 Sample users collection

| _id | email | password |
| --- | --- | --- |
| 1 | rahul@gmail.com | hashed123 |
| 2 | vikash@gmail.com | hashed456 |
| 3 | amit@gmail.com | hashed789 |

The passwords above are illustrative placeholders, not real password hashes.

### 2.2 Unsafe Express route

```js
// ❌ BAD: Do not use this as authentication code.
app.post("/login", async (req, res) => {
  const user = await User.findOne({
    email: req.body.email
  });

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  // BAD: Exposes the user document; no password verification.
  res.json({ user });
});
```

**Problem:** The code never checks whether `req.body.email` is actually a string. It could be an object containing MongoDB query operators.

### 2.3 Case 1 — Normal email input

```json
{ "email": "rahul@gmail.com" }
```

The lookup is equivalent to:

```js
User.findOne({ email: "rahul@gmail.com" });
```

**Result:** Rahul's matching document is returned, as expected.

### 2.4 Case 2 — Malicious object instead of an email

An attacker sends:

```json
{
  "email": {
    "$ne": null
  }
}
```

**What is `$ne`?** It means **not equal** in MongoDB. For example:

```js
User.find({ email: { $ne: null } });
```

This matches documents whose email is not null (and whose email field exists).

If the untrusted object reaches MongoDB as a query selector, the unsafe lookup effectively becomes:

```js
User.findOne({
  email: { $ne: null }
});
```

This no longer means **find Rahul's exact email**. It means **find a user with a non-null email**.

With the sample data, all three users match the condition, but `findOne()` returns **only one matching document**. Without an explicit sort, do not assume which matching user will be returned.

### 2.5 Step-by-step execution flow

```text
Attacker sends { "email": { "$ne": null } }
                   |
                   v
Express parses JSON into req.body.email (an object)
                   |
                   v
Unsafe Mongoose query receives the object
                   |
                   v
email: { $ne: null }  (if interpreted as an operator)
                   |
                   v
Any user with a non-null email can match
                   |
                   v
findOne() returns one matching user, not necessarily the intended one
```

**Why dangerous?** The client changed the **query condition**, not just its value. However, finding a user document **does not automatically bypass authentication**: a correctly implemented login must still verify the password and enforce proper authorization. The demo route is insecure because it does neither and exposes the returned document.

**Mongoose caveat:** Actual behavior depends on the Mongoose version, schema casting, and `sanitizeFilter` settings. The payload may be interpreted as a selector, rejected, or sanitized. Never rely on a particular casting error as your security control.

### 2.6 Fix 1 — Validate the input type

```js
app.post("/users/lookup", async (req, res) => {
  const { email } = req.body ?? {};

  if (typeof email !== "string") {
    return res.status(400).json({ message: "Email must be a string" });
  }

  const user = await User.findOne({ email }).select("name email");
  res.json({ user });
});
```

For the malicious payload, `typeof req.body.email` is `"object"`, so the request gets a **400** response instead of reaching the query. In a real application, also validate email format, length, and authorization.

### 2.7 Fix 2 — Explicit equality and defense in depth

```js
// Use only after validating email as a string.
const user = await User.findOne({
  email: { $eq: email }
});
```

Here the developer chooses the `$eq` (equals) operator rather than allowing the client to choose the query structure. **Type validation is still required.**

Mongoose also provides:

```js
mongoose.set("sanitizeFilter", true);
```

This can help prevent certain nested operator objects from being treated as query selectors, but **does not replace validation or authorization**.

### 2.8 SQL vs NoSQL injection

| SQL injection | MongoDB NoSQL injection |
| --- | --- |
| User input alters SQL instructions | User-controlled objects alter query operators |
| Example: `' OR '1'='1` | Example: `{ "$ne": null }` |
| Defense: parameterized queries | Defense: validated types, controlled filters, sanitization |

**Interview takeaway:** In MongoDB, `{ email: "rahul@gmail.com" }` is an exact-match filter, while `{ email: { $ne: null } }` is an operator-based filter. Never let untrusted client objects choose your database query logic.

## 3. Safe approach: validate and construct the filter

```bash
npm install zod
```

```js
import { z } from "zod";

const lookupSchema = z.strictObject({
  email: z.email(),
});

app.post("/users/lookup", async (req, res) => {
  const parsed = lookupSchema.safeParse(req.body);

  if (!parsed.success) {
    return res.status(400).json({ message: "Invalid email" });
  }

  const user = await User.findOne({
    email: parsed.data.email,
  }).select("name email");

  res.json({ user });
});
```

**Flow:** client sends JSON → Zod requires an email string → controller builds a fixed filter → Mongoose sends the query.

The `email` value is a **validated string**, not a client-controlled query object.

## 4. Never pass the whole request body to a query

Unsafe:

```js
await User.find(req.body);
```

Better: select only allowed filters.

```js
const filterSchema = z.strictObject({
  active: z.boolean().optional(),
});

const parsed = filterSchema.safeParse(req.body);
if (!parsed.success) return res.sendStatus(400);

const filter = {};
if (parsed.data.active !== undefined) {
  filter.active = parsed.data.active;
}

const users = await User.find(filter).select("name email");
```

Now an attacker cannot arbitrarily add `$or`, `$where`, or sensitive fields through the request body.

## 5. Mongoose sanitizeFilter — defense in depth

Mongoose provides a filter-sanitization option:

```js
import mongoose from "mongoose";

mongoose.set("sanitizeFilter", true);
```

This can help defend against query selector injection by wrapping suspicious nested operator objects in `$eq`, but **it does not replace strict input validation or safe query construction**. Understand when your own intended operators are used, and follow Mongoose guidance for trusted filters. Check the Mongoose version you use.

## 6. NoSQL injection vs SQL injection

| SQL injection | MongoDB NoSQL injection |
| --- | --- |
| Untrusted input becomes SQL syntax | Untrusted objects/operators change query semantics |
| Unsafe string-built SQL | Unsafe client-controlled query objects |
| Defense: SQL parameters | Defense: validate types and construct allowlisted filters |

## 7. More safety rules

- Never let the request body supply arbitrary MongoDB filters or update objects.
- Reject unexpected objects when fields should be strings, numbers, or booleans.
- Explicitly allowlist sortable fields, filters, and updateable properties.
- Use authorization checks: safe queries alone do not prevent access to someone else's data.
- Avoid user-controlled `$where` or JavaScript-executing query features.
- Keep Mongoose and MongoDB driver dependencies current.

## Practice

1. What is the problem with `User.findOne({ email: req.body.email })` when `email` can be an object?
2. Why does `z.email()` help in the example?
3. Compare `User.find(req.body)` with `User.find({ active: validatedActive })`.
4. Explain why `sanitizeFilter` is extra defense, not a replacement for validation.

**Interview answer:** NoSQL injection happens when user-controlled data alters a database query through unexpected objects or operators. In MongoDB/Mongoose, defend by validating primitive types, constructing allowlisted filters, restricting updates, and applying defense-in-depth options.

---

## 8. Operator injection vs unsafe updates

MongoDB uses special operators: `$ne` (not equal), `$gt` (greater than), `$or` (logical OR), `$set` (set fields). Operators are useful when **your code** chooses them; dangerous when a client chooses arbitrary operators.

A second risk is **mass assignment** in updates:

```js
// BAD: attacker may submit { role: "admin" }
await User.findByIdAndUpdate(req.user.id, req.body);

// BETTER: validated, explicit allowed fields
const parsed = z.strictObject({
  displayName: z.string().trim().min(1).max(60),
}).safeParse(req.body);
if (!parsed.success) return res.sendStatus(400);

await User.updateOne(
  { _id: req.user.id },
  { $set: { displayName: parsed.data.displayName } }
);
```

NoSQL injection and mass assignment are different problems, but both happen when the API trusts client-controlled object structure.

## 9. Validate ObjectId and avoid client-controlled queries

```js
import mongoose from "mongoose";

app.get("/users/:id", requireAuth, async (req, res) => {
  const id = req.params.id;
  if (!mongoose.isObjectIdOrHexString(id)) {
    return res.status(400).json({ message: "Invalid ID" });
  }
  if (id !== req.user.id) return res.sendStatus(403);

  const user = await User.findById(id).select("name email");
  if (!user) return res.sendStatus(404);
  res.json({ user });
});
```

This is a self-profile example: a valid ID alone does not grant access. The authenticated identity must come from verified auth middleware.

## 10. Query selector injection and Mongoose behavior

A user might send a JSON value like `{"email":{"$ne":null}}`. Depending on the API, casting, and Mongoose settings, that object may be rejected or interpreted as a selector. **Never rely on a particular casting error to protect you.**

Safer pattern:
1. Validate `email` as a string.
2. Build `{ email: validatedEmail }` in server code.
3. Do not accept arbitrary `filter`, `sort`, `projection`, or `update` objects from the client.
4. Consider `sanitizeFilter` as extra defense, not a substitute for the above.

## 11. Regex and aggregation pitfalls

- Avoid building arbitrary regular expressions from user input. Regex patterns can be expensive or change matching behavior. For literal text search, escape regex metacharacters and limit length, or use appropriate indexed search features.
- Never pass user-controlled arrays directly into `Model.aggregate(req.body.pipeline)`; aggregation stages can expose data or cause expensive queries.
- Do not allow user-selected `$where` or JavaScript execution features.

Example of safe fixed aggregation shape:

```js
const parsed = z.strictObject({
  status: z.enum(["paid", "pending"]),
}).safeParse(req.query);
if (!parsed.success) return res.sendStatus(400);

const summary = await Order.aggregate([
  { $match: { status: parsed.data.status, userId: req.user._id } },
  { $group: { _id: "$status", count: { $sum: 1 } } },
]);
```

`req.user._id` must be a trusted, properly typed ID from verified authentication.

## 12. Preventing abuse beyond injection

- Set sensible query limits, timeouts, and indexes.
- Keep MongoDB credentials least-privileged.
- Never return raw database errors or private fields.
- Test unexpected arrays/objects in fields meant to be strings.
- Ensure authorization applies to every resource and query.

**Interview takeaway:** Mongoose schemas help, but only strict input types, allowlisted query structure, permission checks, and defense in depth make API queries trustworthy.