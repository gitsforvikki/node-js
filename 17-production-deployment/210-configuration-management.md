# Lesson 210 — Configuration Management

## 1. What is configuration?
Configuration controls how the **same application code** behaves in different environments: port, database URL, logging level, allowed origin, session secret and external API endpoints.

**Code is committed; secrets are not.** Configuration is validated at startup, not scattered as `process.env.X` everywhere.

## 2. Typical files
```text
.env                # local secrets; ignored by Git
.env.example        # safe empty template; committed
src/config/env.js   # typed/validated configuration
```
`.gitignore`:
```
.env
.env.*
!.env.example
```

## 3. Validate environment with Zod
Install `npm install zod`. In `src/config/env.js`:
```js
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  MONGODB_URI: z.string().min(1),
  SESSION_SECRET: z.string().min(32),
  CORS_ORIGIN: z.url(),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export const config = envSchema.parse(process.env);
```

Use `config.MONGODB_URI` instead of rereading unvalidated environment variables in every module. The schema validates shape, but it doesn't prove DB credentials actually work.

## 4. Load env before importing configuration
For supported Node versions:
```bash
node --env-file=.env src/server.js
```
Or use `dotenv` at the application's entry point **before** modules read config. Beware of ESM static import evaluation: loading dotenv in a later statement can be too late.

Production hosting typically injects environment variables; don't require a physical `.env` file in containers.

## 5. Public vs private variables
In a Next.js frontend, `NEXT_PUBLIC_*` is intended to be embedded in client bundles. Never set `NEXT_PUBLIC_SESSION_SECRET` or put database passwords into frontend code.

Keep dev and production credentials separate, restrict access, use rotation and avoid logging complete configuration.

## 6. Feature flags vs config
Configuration describes environment differences (DB URL, port). Feature flags enable or disable behavior (new checkout flow). Dynamic flags require careful caching/fallback strategies and shouldn't bypass security/authorization.

## 7. Common mistakes
Missing required secrets; using a real production URI during tests; defaulting to an insecure JWT key; checking `.env` into Git; printing `process.env`; hardcoding CORS origins for all environments.

## Practice
Create `.env.example`, enforce required `MONGODB_URI` and `SESSION_SECRET`, and verify startup fails immediately with a clear variable name when missing.

**Interview answer:** Configuration management separates environment-specific values from code. Validate inputs at startup, store secrets securely, apply least privilege and keep consistent configurations across deployment environments.
