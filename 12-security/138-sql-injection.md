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
