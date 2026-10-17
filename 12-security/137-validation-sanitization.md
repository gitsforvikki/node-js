# Lesson 137 — Input Validation & Sanitization

## 1. Why validate input?

**Never assume the frontend sends correct data.** Anyone can call your API using Postman, curl, or their own script.

**Validation** means checking if data follows your rules: `age` must be a number, `email` must be valid, and `name` must not be empty.

**Sanitization/normalization** means safely preparing allowed input, such as trimming spaces or lowercasing an email. It is **not** a magic function that makes every string safe everywhere.

## 2. Simple difference

Input:

```json
{ "name": "  Vikash  ", "email": "VIKASH@EXAMPLE.COM", "age": "hello" }
```

- **Validation:** reject `age: "hello"` because age is not a number.
- **Normalization:** change `"  Vikash  "` to `"Vikash"`, if your business rules allow it.
- **Output encoding:** when inserting untrusted text into HTML, the frontend must handle it safely. Trimming or lowercasing does not prevent XSS.

## 3. Use Zod with Express 5

```bash
npm install express zod
```

```js
import express from "express";
import { z } from "zod";

const app = express();
app.use(express.json({ limit: "16kb" }));

const registerSchema = z.strictObject({
  name: z.string().trim().min(2).max(60),
  email: z.email().trim().toLowerCase(),
  age: z.number().int().min(18).max(120),
});

app.post("/register", (req, res) => {
  const result = registerSchema.safeParse(req.body);

  if (!result.success) {
    return res.status(400).json({
      message: "Invalid input",
      errors: result.error.flatten().fieldErrors,
    });
  }

  // Use parsed data, NOT the original req.body.
  const { name, email, age } = result.data;
  res.status(201).json({ name, email, age });
});
```

**Note:** This uses current Zod 4 style (`z.email()`, `z.strictObject()`). On a different major version, verify the API in its documentation.

### What happens when a request arrives?

```text
req.body
  |
  v
safeParse(req.body)
  |
  +--> invalid -> 400 response
  |
  +--> valid -> result.data -> controller/service
```

`safeParse()` returns a result object; it does not throw for ordinary validation failures.

## 4. Never use input directly as a database query

Bad idea:

```js
await User.findOne(req.body);
```

Better:

```js
const result = z.strictObject({
  email: z.email(),
}).safeParse(req.body);

if (!result.success) {
  return res.status(400).json({ message: "Invalid input" });
}

const user = await User.findOne({ email: result.data.email });
```

Build the allowed filter yourself. Do not let clients choose database operators or unexpected fields.

## 5. Validate params, query strings, and body separately

- `req.params`: route IDs, such as `/users/:id`.
- `req.query`: filters, pagination, sorting (usually strings).
- `req.body`: submitted JSON.

Example:

```js
const paginationSchema = z.strictObject({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

const result = paginationSchema.safeParse(req.query);
if (!result.success) return res.sendStatus(400);

const { page, limit } = result.data;
```

**Caution:** Coercion is a deliberate choice; don't coerce sensitive or security-related fields without understanding how values convert.

## 6. Validation vs escaping vs parameterization

| Defense | Best use |
| --- | --- |
| Validation | Reject unexpected types, lengths, values, extra keys |
| Normalization | Trim, lowercase, convert formats intentionally |
| SQL parameters | Keep user values separate from SQL code |
| Allowlisted MongoDB filters | Keep operators under your control |
| HTML output encoding | Prevent text from becoming executable HTML |

**Important:** Even perfectly valid strings can contain punctuation used in attacks. Validation is not a replacement for parameterized SQL, permissions checks, or safe HTML rendering.

## 7. Common mistakes

- Checking input in React only; attackers can bypass React.
- Validating `req.body`, then accidentally using the *unvalidated* body.
- Using permissive schemas where extra fields silently change permissions.
- Relying on a generic 'sanitize everything' package.
- Returning raw passwords or secrets in validation errors/logs.

## Practice

1. Try `age: "twenty"`, `age: 17`, and `age: 21`. Which passes?
2. Send `role: "admin"` to `/register`. Why should the API reject or ignore it rather than save it?
3. Add a schema for `POST /products` with `name`, `price`, `stock`.
4. Explain why validation alone cannot stop SQL injection.

**Interview answer:** Validate on the backend at the trust boundary, use only parsed/allowlisted fields, normalize when appropriate, and apply context-specific protections such as SQL parameters and HTML output encoding.

---

## 8. Validate every untrusted entry point

The body is only one source. Attackers also control `req.params`, `req.query`, request headers, cookies, uploaded files, and sometimes data read from external services.

**Do not confuse these operations:**
- **Validation:** accept/reject (age 18–120).
- **Normalization:** make an allowed representation consistent (trim whitespace).
- **Sanitization:** remove/neutralize specific unwanted content **for a known purpose**.
- **Encoding/escaping:** represent text safely in its output context (HTML, URL, SQL parameters). Different contexts need different protection.

Example: A blog comment can legitimately contain `<` or quotes. You don't need to delete them from the database. Render it as **text**, not executable HTML.

## 9. Reusable validation middleware (Express 5)

```js
const validateBody = (schema) => (req, res, next) => {
  const result = schema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({
      message: "Invalid request",
      errors: result.error.flatten().fieldErrors,
    });
  }
  res.locals.validatedBody = result.data;
  next();
};

const updateSchema = z.strictObject({
  displayName: z.string().trim().min(2).max(50),
});

app.patch("/me", requireAuth, validateBody(updateSchema), async (req, res) => {
  const { displayName } = res.locals.validatedBody;
  await User.updateOne({ _id: req.user.id }, { $set: { displayName } });
  res.sendStatus(204);
});
```

**Why `res.locals`?** We keep parsed/normalized data separate from the original untrusted `req.body`. This middleware checks **shape**, not whether the caller is allowed to edit the record; authorization is a separate step.

## 10. Field rules and cross-field rules

Use `min`, `max`, `int`, `enum`, and optional fields to express business constraints. For related fields, use `refine`:

```js
const passwordChange = z.strictObject({
  newPassword: z.string().min(12).max(128),
  confirmPassword: z.string(),
}).refine(
  (data) => data.newPassword === data.confirmPassword,
  { message: "Passwords do not match", path: ["confirmPassword"] }
);
```

Do not echo submitted passwords in error responses or logs. A good validation message says *what rule failed*, not the secret value.

## 11. Update APIs and allowlists

A `PATCH` body is usually **partial**: use an explicit update schema with optional allowed fields, then apply only those keys. Never spread the entire body into `$set`.

```js
const patchSchema = z.strictObject({
  name: z.string().trim().min(2).optional(),
  bio: z.string().max(500).optional(),
});
const parsed = patchSchema.safeParse(req.body);
if (!parsed.success) return res.sendStatus(400);
await User.updateOne({ _id: req.user.id }, { $set: parsed.data });
```

Add a check to reject empty patches if your API requires at least one changed field. Database constraints, unique indexes, and permission checks are **still necessary**.

## 12. HTML sanitization is a separate decision

If you support **rich HTML** from users, use a maintained HTML sanitizer configured with an allowlist of tags/attributes **before rendering HTML**. If you only need plain text, prefer rendering text normally (React escapes text by default) and avoid `dangerouslySetInnerHTML`.

Never try to prevent XSS by deleting only `<script>`; many browser features can execute script. See Lesson 140.

## Practice extension

Create `POST /products` validation for `name` (2–80 chars), `price` (nonnegative number), `stock` (nonnegative integer), and reject an unexpected `isAdmin` field. Then test missing fields, wrong types, and extra keys.