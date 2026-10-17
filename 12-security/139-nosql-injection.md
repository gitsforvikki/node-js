# Lesson 139 — NoSQL Injection

## 1. What is NoSQL injection?

NoSQL injection happens when untrusted input changes the **meaning of a NoSQL database query**.

MongoDB queries are JavaScript-like objects. Some keys, such as `$ne`, `$gt`, and `$or`, are **operators** rather than ordinary field names. Allowing a user to send arbitrary query objects is dangerous.

**Remember:** SQL injection targets SQL query text; MongoDB operator injection commonly abuses unexpected objects and operators.

## 2. Unsafe login lookup example

```js
// BAD: req.body.email could be an object instead of a string.
const user = await User.findOne({ email: req.body.email });
```

A malicious client could send a JSON object instead of an email string:

```json
{
  "email": { "$ne": null }
}
```

`$ne` means "not equal". If an untrusted object reaches a MongoDB query as an operator, it can change the search from "match this email" to something else.

**Important:** Actual behavior depends on Mongoose version, schema casting, and sanitization options. Never depend on accidental rejection as your primary defense.

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
