# Lesson 146 — Secrets Management in Node.js

## 1. What is a secret?

A **secret** is information that gives access to something valuable. Examples:
- MongoDB/PostgreSQL passwords and connection strings
- JWT signing keys and session secrets
- Stripe/Razorpay secret API keys
- Email-provider credentials
- Private keys and cloud service credentials

If a secret leaks, an attacker may impersonate your server or access services/data.

## 2. Environment variables vs hardcoded secrets

**Bad:**

```js
const mongoUri = "mongodb+srv://admin:real-password@cluster.example/db";
```

**Better:**

```js
const mongoUri = process.env.MONGODB_URI;
```

Environment variables help keep secrets out of source code, but they are **not automatically encrypted or safe**. They can leak through logs, debug output, crash reports, process inspection, or misconfigured hosting.

## 3. Local development with .env

```text
project/
  .env
  .env.example
  .gitignore
  src/
    server.js
```

`.env` (local only):

```dotenv
MONGODB_URI=mongodb+srv://USER:PASSWORD@HOST/shophub
SESSION_SECRET=replace-with-long-random-secret
PORT=3000
```

`.env.example` (safe to commit):

```dotenv
MONGODB_URI=
SESSION_SECRET=
PORT=3000
```

`.gitignore`:

```gitignore
.env
.env.*
!.env.example
```

Node.js supports loading env files via `node --env-file=.env src/server.js` on supported versions. Or use `dotenv` with an early import:

```js
import "dotenv/config";
```

**Important:** Load environment variables **before** modules that read them at import time.

## 4. Validate secrets at startup

```js
const required = ["MONGODB_URI", "SESSION_SECRET"];

for (const name of required) {
  if (!process.env[name]) {
    throw new Error(`Missing required environment variable: ${name}`);
  }
}
```

Failing early is better than starting a server with missing signing keys or database credentials.

## 5. Express + MongoDB example

```js
import "dotenv/config";
import mongoose from "mongoose";
import app from "./app.js";

async function start() {
  if (!process.env.MONGODB_URI) {
    throw new Error("MONGODB_URI missing");
  }

  await mongoose.connect(process.env.MONGODB_URI);
  app.listen(Number(process.env.PORT || 3000), () => {
    console.log("Server ready");
  });
}

start().catch((error) => {
  console.error("Startup failed:", error.message);
  process.exit(1);
});
```

**Flow:** load env → validate config → connect database → start HTTP server. Avoid printing full database connection strings or exception objects that may contain secrets.

## 6. Frontend vs backend secrets (Next.js example)

**Never place private keys in frontend code.** Browser JavaScript is visible to users.

In Next.js, variables prefixed with `NEXT_PUBLIC_` are intended to be exposed to the browser build. Do not prefix server secrets this way.

```text
NEXT_PUBLIC_APP_NAME=ShopHub       # Public configuration
MONGODB_URI=...                    # Server only
STRIPE_SECRET_KEY=...              # Server only
```

Even server-side values can leak if returned in API responses, logged, or bundled incorrectly.

## 7. Production secret storage

For deployed apps, use the hosting provider's secret/environment-variable settings or a dedicated secret manager (e.g., AWS Secrets Manager, Google Secret Manager, Azure Key Vault, Vault).

Typical flow:

```text
Developer -> authorized secret manager
                    |
                    v
             Deployment/runtime
                    |
                    v
               Node.js server
```

Protect access using roles, auditing, and least privilege. Prefer short-lived workload identity/credentials where supported instead of long-lived static keys.

## 8. Secret rotation

**Rotation** means replacing a credential with a new one.

Example for a leaked database password:
1. Create a new credential with minimum permissions.
2. Update deployment secret/config.
3. Restart/redeploy or reload connections as needed.
4. Verify the application works.
5. Revoke the old credential.
6. Check logs and audit access for misuse.

For JWT signing keys, plan for a **transition period** (e.g., key IDs and multiple verification keys) or intentional session invalidation. Blindly changing a key may log out all users.

## 9. What if you accidentally commit a secret?

**Deleting the line is not enough**: it remains in Git history and may have been copied.

1. Immediately **revoke/rotate** the exposed credential.
2. Check usage/audit logs.
3. Remove it from current code and relevant history where practical.
4. Notify appropriate owners if data may have been accessed.
5. Add automated secret scanning/pre-commit checks.

History cleanup does **not** undo exposure. Rotation is the critical step.

## 10. Common mistakes

- Committing `.env` or sending it in chat/screenshots.
- Printing `process.env` or credentials in logs.
- Reusing one secret across development and production.
- Giving a database account admin permissions when only read/write is needed.
- Putting server secrets in `NEXT_PUBLIC_*`.
- Assuming an env variable is an encrypted vault.
- Keeping leaked keys active after deleting them from Git.

## 11. Practice

1. Create `.env.example` without real credentials.
2. Add startup validation for `MONGODB_URI` and `SESSION_SECRET`.
3. Explain the difference between public configuration and private credentials.
4. Write a rotation plan for a leaked database password.
5. Verify `.env` is ignored using `git check-ignore -v .env`.

**Interview answer:** Secrets management is securely storing, granting access to, using, rotating, and revoking sensitive credentials. Keep secrets out of code and browser bundles, use least privilege and managed secret storage in production, validate config at startup, and rotate exposed credentials immediately.
