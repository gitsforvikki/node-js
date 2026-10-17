# Lesson 138 — SQL Injection

## 1. What is SQL injection?

SQL injection happens when **untrusted input changes the meaning of a SQL query**.

Imagine a login search that builds SQL by joining strings. Someone enters SQL-like text instead of a normal email; the resulting query may behave differently than the developer intended.

**Main idea:** User input should be treated as **data**, never as part of SQL instructions.

## 2. Unsafe example — do NOT use in an app

```js
// BAD: untrusted input is placed directly into SQL text.
const sql = "SELECT id, email FROM users WHERE email = '" +
  req.body.email + "'";

const result = await pool.query(sql);
```

**Why dangerous?** The email value can contain characters interpreted as SQL syntax. The database cannot reliably separate the developer's SQL from user-provided text.

## 3. Correct approach — parameterized query

Using `pg` (node-postgres) with PostgreSQL:

```bash
npm install pg
```

```js
import pg from "pg";

const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
});

const email = req.body.email; // Validate shape/type first.

const result = await pool.query(
  "SELECT id, email FROM users WHERE email = $1",
  [email]
);

console.log(result.rows);
```

**How it works:**

1. SQL structure is fixed: `WHERE email = $1`.
2. `[email]` supplies the value separately.
3. The driver/database treats it as data, **not executable SQL**.
4. The query safely searches for that email.

## 4. Express 5 route example

```js
import express from "express";
import { z } from "zod";

const app = express();
app.use(express.json());

const bodySchema = z.strictObject({ email: z.email() });

app.post("/users/search", async (req, res) => {
  const parsed = bodySchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ message: "Invalid email" });
  }

  const result = await pool.query(
    "SELECT id, email FROM users WHERE email = $1",
    [parsed.data.email]
  );

  res.json({ users: result.rows });
});
```

Prerequisites: configure a valid PostgreSQL `DATABASE_URL`, create a `users` table, and initialize the `pool` shown above. Express 5 forwards async route rejections to error middleware.

## 5. What about dynamic sorting and table names?

SQL placeholders such as `$1` work for **values**, not SQL identifiers like table names or column names.

For sorting, choose from a **fixed allowlist**:

```js
const allowedSort = {
  newest: "created_at DESC",
  oldest: "created_at ASC",
};

const orderBy = allowedSort[req.query.sort] ?? allowedSort.newest;
const result = await pool.query(
  `SELECT id, email FROM users ORDER BY ${orderBy} LIMIT $1`,
  [20]
);
```

This is safe because the SQL fragment comes from **hardcoded developer-controlled choices**, never directly from the request.

## 6. Extra protection

- Use the database account with only necessary permissions.
- Do not show raw SQL errors to clients.
- Validate input, but **never rely only on validation** against injection.
- Prefer parameterized queries, or trusted query builders/ORMs that parameterize values.
- Keep database libraries updated.

## Practice

1. Explain in one sentence why string concatenation in SQL is risky.
2. Convert this unsafe query to a parameterized one:
   `"SELECT * FROM products WHERE id = " + req.params.id`
3. Why can't you parameterize a column name using `$1`?
4. What is the purpose of the `[email]` array in `pool.query(sql, [email])`?

**Interview answer:** SQL injection is when untrusted input becomes SQL instructions. Parameterized queries prevent this by separating query structure from values; validation and least-privilege database permissions provide additional protection.

---

## 7. Understand the attack with a simple mental model

Suppose a developer builds `WHERE email = 'USER_INPUT'` by concatenating strings. An attacker can include SQL punctuation so the database interprets some input as **instructions**, not as the email value. This can change filtering or expose unauthorized rows.

With `WHERE email = $1`, the database receives a fixed query and a separate value. Even unusual punctuation is compared as **data**.

**Key point:** Escaping quotes manually is error-prone. Parameter binding is the default defense.

## 8. Parameterize UPDATE, INSERT, DELETE too

```js
// Safe INSERT
await pool.query(
  "INSERT INTO users (email, name) VALUES ($1, $2)",
  [email, name]
);

// Safe UPDATE, also scoped to authenticated owner
await pool.query(
  "UPDATE users SET name = $1 WHERE id = $2",
  [name, req.user.id]
);

// Safe DELETE
await pool.query(
  "DELETE FROM sessions WHERE user_id = $1 AND id = $2",
  [req.user.id, sessionId]
);
```

Injection safety and **authorization** are different: a parameterized query can still return another user's data if you forget ownership checks.

## 9. LIKE searches and wildcard characters

Parameterized SQL prevents injection, but `%` and `_` still act as **LIKE wildcards**. If the user intends a literal search, escape wildcard characters too:

```js
const escaped = term.replace(/[\\%_]/g, "\\$&");
const result = await pool.query(
  "SELECT id, name FROM products WHERE name ILIKE $1 ESCAPE '\\'",
  [`%${escaped}%`]
);
```

If wildcard search is intended, document it and bound input length/result count.

## 10. Transactions, ORMs, and stored procedures

- **Transactions** make related writes atomic; they do **not** automatically prevent injection.
- **ORM/query builders** are safe when they bind values correctly; unsafe raw SQL string interpolation can still introduce injection.
- **Stored procedures** are not automatically safe if they construct dynamic SQL with untrusted strings.

Example with a query builder: use parameterized expressions, not user-controlled raw SQL snippets.

## 11. Second-order SQL injection

Sometimes unsafe input is **stored first** and later used to build a query. Being saved in a database does not make a string trustworthy. Always parameterize it when using it in SQL later.

## 12. How to test your own API

- Send quotes and unusual punctuation in search terms; they should be treated as data or rejected by validation.
- Check the API does not expose raw SQL errors.
- Verify database user has only required table permissions.
- Test sort-field allowlists and search pagination.

**Interview takeaway:** Injection can occur in SELECT, UPDATE, INSERT, DELETE, raw ORM queries, and later queries built from stored data. Bind **values**; allowlist **identifiers and SQL fragments**.