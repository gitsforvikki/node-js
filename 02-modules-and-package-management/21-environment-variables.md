# Lesson 21 — Environment Variables and .env

## What you'll learn

- Why applications should separate configuration from source code
- How `process.env` works in Node.js
- What a `.env` file is used for
- How to handle secrets safely
- How production systems manage environment-specific configuration
- Common mistakes and interview questions

## Core Concept

A professional application should not hardcode environment-specific values directly into the source code.

Bad:

```js
const DATABASE_URL = "postgres://admin:password@prod-db.example.com/app";
const JWT_SECRET = "super-secret";
```

Better:

```js
const databaseUrl = process.env.DATABASE_URL;
const jwtSecret = process.env.JWT_SECRET;
```

This separates:

```text
Application Code
      |
      +--> business logic
      +--> controllers
      +--> services
      +--> repositories

Environment Configuration
      |
      +--> database URL
      +--> secrets
      +--> ports
      +--> API endpoints
      +--> environment name
```

## What is process.env?

Node.js exposes environment variables through:

```js
process.env
```

Example:

```js
console.log(process.env.NODE_ENV);
console.log(process.env.PORT);
```

You can provide a variable from the shell:

```bash
PORT=5000 node app.js
```

Then:

```js
console.log(process.env.PORT);
```

returns:

```text
5000
```

## Important: Environment Variables Are Strings

This is a common production bug.

```bash
PORT=5000
FEATURE_ENABLED=false
```

In Node.js:

```js
console.log(typeof process.env.PORT);
// string

console.log(Boolean(process.env.FEATURE_ENABLED));
// true
```

Why is `"false"` truthy?

Because it is still a non-empty string.

Correct conversion:

```js
const port = Number(process.env.PORT ?? 3000);

const featureEnabled =
  process.env.FEATURE_ENABLED === "true";
```

## What is a .env File?

A `.env` file is a convenient way to store local environment variables.

Example:

```env
PORT=3000
NODE_ENV=development
DATABASE_URL=postgres://localhost:5432/myapp
JWT_SECRET=local-development-secret
```

Your application can load these values and access them through `process.env`.

Historically, the popular `dotenv` package has been used for this purpose.

Modern Node.js also provides built-in environment-file support in supported versions, so choose an approach that matches the Node.js version and project conventions.

## Example with dotenv

Install:

```bash
npm install dotenv
```

Then:

```js
import "dotenv/config";

console.log(process.env.PORT);
```

Another explicit style:

```js
import dotenv from "dotenv";

dotenv.config();
```

## Modern Built-in Node.js Approach

Modern Node.js versions can load environment files from the command line.

Example:

```bash
node --env-file=.env src/server.js
```

Then:

```js
console.log(process.env.DATABASE_URL);
```

This can remove the need for a third-party loader in simple applications.

## Never Commit Real Secrets

Your `.gitignore` should normally contain:

```gitignore
.env
.env.local
.env.*.local
```

Never commit values such as:

- database passwords
- payment secrets
- JWT secrets
- OAuth client secrets
- private API keys
- cloud credentials

## Use .env.example

A good repository should show which variables are required without exposing actual values.

Commit this:

```env
# .env.example

PORT=
NODE_ENV=
DATABASE_URL=
JWT_SECRET=
REDIS_URL=
FRONTEND_URL=
```

Do not commit this:

```text
.env
```

This makes onboarding easier:

```text
Developer clones repository
        |
        v
reads .env.example
        |
        v
creates own .env
        |
        v
application starts
```

## Validate Environment Variables at Startup

A weak approach:

```js
const databaseUrl = process.env.DATABASE_URL;
```

If it is missing, the app may fail much later with a confusing database error.

A better approach is to fail immediately.

```js
const requiredVariables = [
  "DATABASE_URL",
  "JWT_SECRET",
];

for (const key of requiredVariables) {
  if (!process.env[key]) {
    throw new Error(
      `Missing required environment variable: ${key}`
    );
  }
}
```

Production applications often use a validation library such as Zod, Joi, or another schema validator.

## Industry-Ready Configuration Pattern

Avoid reading `process.env` throughout the entire application.

Instead:

```text
Environment Variables
        |
        v
Configuration Layer
        |
        v
Validation + Type Conversion
        |
        v
Validated Config Object
        |
        +--> Database
        +--> Auth
        +--> Redis
        +--> Server
        +--> External Services
```

Example:

```js
// config/env.js

function requireEnv(name) {
  const value = process.env[name];

  if (!value) {
    throw new Error(`Missing environment variable: ${name}`);
  }

  return value;
}

export const env = {
  port: Number(process.env.PORT ?? 3000),
  nodeEnv: process.env.NODE_ENV ?? "development",
  databaseUrl: requireEnv("DATABASE_URL"),
  jwtSecret: requireEnv("JWT_SECRET"),
};
```

Elsewhere:

```js
import { env } from "./config/env.js";

console.log(env.port);
```

Now configuration is centralized.

## Development, Staging and Production

Different environments need different configuration.

```text
Development
├── localhost database
├── verbose logs
└── local services

Staging
├── staging database
├── production-like integrations
└── testing credentials

Production
├── production database
├── secure secrets
├── strict CORS
└── production service endpoints
```

The source code can remain mostly identical.

Only configuration changes.

## Production Secret Management

A local `.env` file is convenient for development, but production commonly uses:

- deployment-platform environment variables
- Jenkins credentials
- GitHub Actions secrets
- Docker secrets
- Kubernetes Secrets
- AWS Secrets Manager
- Google Secret Manager
- Azure Key Vault
- HashiCorp Vault

Mental model:

```text
Source Code Repository
        |
        X
   no secrets here

Secret Store
        |
        v
Deployment Environment
        |
        v
process.env
        |
        v
Node.js Application
```

## NODE_ENV

Common values:

```text
development
test
production
```

Example:

```js
const isProduction =
  process.env.NODE_ENV === "production";
```

It may be used for:

- logging level
- debugging behavior
- cookie security configuration
- performance-related setup

Do not use environment checks to duplicate large amounts of business logic.

## Security Mistakes

### 1. Committing .env

Once a secret enters Git history, simply deleting the file later may not remove the secret from history.

The secret should be rotated.

### 2. Logging process.env

Never do:

```js
console.log(process.env);
```

in production.

It may expose every secret available to the process.

### 3. Sending secrets to clients

Backend secrets should never be returned through APIs or embedded in frontend bundles.

### 4. Sharing production secrets with development

Use different credentials for each environment.

### 5. Trusting missing configuration

Fail fast when required configuration is missing.

## Practical Production Example

```js
// config/index.js

const required = (key) => {
  const value = process.env[key];

  if (!value) {
    throw new Error(`Missing config: ${key}`);
  }

  return value;
};

export const config = {
  app: {
    port: Number(process.env.PORT ?? 3000),
    env: process.env.NODE_ENV ?? "development",
  },

  database: {
    url: required("DATABASE_URL"),
  },

  auth: {
    jwtSecret: required("JWT_SECRET"),
  },

  redis: {
    url: process.env.REDIS_URL,
  },
};
```

Application code becomes clean:

```js
import { config } from "./config/index.js";

server.listen(config.app.port);
```

## Common Mistakes

- committing `.env`
- assuming environment variables have number/boolean types
- accessing `process.env` everywhere
- using production secrets locally
- exposing secrets in error messages
- not validating configuration at startup
- hardcoding service URLs
- confusing frontend public environment variables with backend secrets

## Interview Questions

### Why do we use environment variables?

To separate environment-specific configuration and secrets from application source code.

### What is process.env?

It is Node.js's interface for accessing environment variables available to the running process.

### Are environment variable values automatically converted to numbers or booleans?

No. They are generally represented as strings and should be explicitly parsed and validated.

### Should .env be committed to Git?

A real secret-containing `.env` file should generally not be committed. A sanitized `.env.example` should be committed instead.

### Why validate environment variables during startup?

Failing fast gives clear configuration errors before the application begins serving traffic.

## Interview-Ready Summary

```text
Environment Variables
        |
        v
process.env
        |
        v
Validation
        |
        v
Central Config Object
        |
        v
Application
```

Use `.env` primarily as a convenient development configuration source.

Keep real secrets out of Git.

In production, use secure deployment or secret-management systems.

## Practice Task

Create:

```text
project/
├── src/
│   ├── config/
│   │   └── env.js
│   └── server.js
├── .env
├── .env.example
└── .gitignore
```

Requirements:

1. Load `PORT`, `DATABASE_URL`, and `JWT_SECRET`
2. Convert `PORT` to a number
3. Fail during startup when required configuration is missing
4. Ignore `.env` in Git
5. Commit only `.env.example`

## Section 2 Final Mental Model

You should now see Node.js project management like this:

```text
Application
   |
   +--> Modules
   |      |
   |      +--> CommonJS / ESM
   |      +--> Resolution
   |
   +--> package.json
   |      |
   |      +--> metadata
   |      +--> scripts
   |      +--> dependency ranges
   |
   +--> package-lock.json
   |      |
   |      +--> exact dependency graph
   |
   +--> npm
   |      |
   |      +--> install
   |      +--> npm ci
   |      +--> audit
   |
   +--> Environment Configuration
          |
          +--> process.env
          +--> validation
          +--> secret management
```

These concepts are the foundation for reproducible builds, CI/CD, package security, scalable project organization, and production Node.js applications.
