# Lesson 219 — Environment-specific Configuration

## 1. One codebase, multiple environments
Development, test, staging and production should run the **same application logic**, with different database URLs, secrets, CORS origins, log levels and external providers.

```text
Same source/image
   -> dev: local DB / debug logging
   -> test: isolated DB / fake providers
   -> staging: production-like test services
   -> prod: managed DB / restricted credentials
```

## 2. Validate with Zod
```js
import { z } from "zod";

const schema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  MONGODB_URI: z.string().min(1),
  SESSION_SECRET: z.string().min(32),
  CORS_ORIGIN: z.url(),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export const config = schema.parse(process.env);
```

`NODE_ENV` identifies runtime mode but **doesn't automatically choose safe credentials**. Always validate the actual configuration before starting the app.

## 3. Development example
`.env.development` (local, ignored by Git):
```dotenv
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/shop_dev
CORS_ORIGIN=http://localhost:3000
SESSION_SECRET=local-development-secret-change-me-32
LOG_LEVEL=debug
```

Production config should come from your deployment's secret/configuration settings, not committed `.env.production`. Keep staging and production DBs separate.

## 4. Environment loading
Supported Node versions can use `node --env-file=.env.development src/server.js`. In hosted platforms, inject values at runtime. Ensure env values load **before** modules read them.

Build once and promote the same Docker image across environments. **Exception:** browser bundles may bake build-time variables into code; design frontend runtime config separately if needed.

## 5. Configuration vs secrets vs feature flags
- Config: PORT, hostname, logging level.
- Secret: database password, session signing key, private API key.
- Feature flag: turns a feature on/off, perhaps dynamically.

Don't expose server secrets using Next.js `NEXT_PUBLIC_*`. Rotate secrets; do not print `process.env` in logs.

## 6. Test environment isolation
Integration tests must **never** use production data. Add a startup/test guard rejecting known production URI patterns when `NODE_ENV=test`, and use isolated credentials and accounts.

## 7. Common mistakes
Using prod DB in tests; committing `.env`; env-variable defaults with weak JWT secrets; differing workers/API env configurations; logging credentials; assuming changing an env var automatically changes a running process.

## Practice
Run the same API image against two different isolated databases and verify no data crossover. Confirm startup fails when `SESSION_SECRET` is missing.

**Interview answer:** Environment-specific configuration separates values from code. Validate them at startup and use separate secrets/resources per environment while promoting one tested artifact.
