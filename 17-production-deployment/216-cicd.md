# Lesson 216 — CI/CD for Node.js

## 1. What is CI/CD?
**Continuous Integration (CI)** checks code changes automatically: install dependencies, lint, test, build. **Continuous Delivery** produces a deployable artifact that may need approval. **Continuous Deployment** automatically releases a passing artifact.

```text
Git push -> CI: npm ci -> lint -> tests -> build
         -> CD: publish versioned image -> staging -> checks -> production
```

Real example: a developer changes the ShopHub checkout controller. CI should reject the change if order tests fail, before customers see it.

## 2. GitHub Actions example
Prerequisites: `package-lock.json` committed and `lint`, `test`, `build` scripts defined in package.json.

`.github/workflows/ci.yml`:
```yaml
name: Node CI
on:
  push:
    branches: [main]
  pull_request:
permissions:
  contents: read
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
      - run: npm audit --audit-level=high
```

If the project is plain JavaScript with no build step, remove `npm run build`. For high-security production, pin third-party Actions to reviewed commit hashes and review supply-chain risks.

## 3. How it executes
When a commit/PR triggers the workflow, GitHub provisions a runner, checks out source, installs exact lockfile dependencies with `npm ci`, runs checks, and marks the job failed on a nonzero exit. Later deployment jobs should depend on successful verification.

## 4. Deploy one immutable build
A release pipeline can build `registry.example/node-api:COMMIT_SHA`, publish it to a registry, then deploy the **same image** in staging and production. Do not silently rebuild different code for each environment. Use an image digest for strong artifact identity.

## 5. Secure deployment
- Keep production credentials out of untrusted PR jobs.
- Use environment approvals, branch protections and short-lived cloud credentials/OIDC where supported.
- Run smoke checks after deploy and monitor HTTP 5xx rate and latency.
- Plan rollback. Database schema changes may not be reversible.

## 6. Database migrations
Use **expand → migrate → contract**: add backward-compatible fields, deploy code that supports old/new forms, backfill, and remove old fields only when all instances have switched.

## 7. Strategies
**Rolling:** replace instances gradually. **Blue-green:** prepare parallel old/new environments and switch traffic. **Canary:** route limited traffic to new release before scaling up. All need readiness checks and monitoring.

## 8. Common mistakes
Deploying directly from untrusted PRs; storing registry passwords in YAML; skipping tests; using untraceable latest tags; making destructive DB changes before old replicas exit.

## Practice
Create the workflow above in a test repository, intentionally fail a unit test, then fix it. Explain CI vs continuous delivery vs continuous deployment.

**Interview answer:** CI verifies changes automatically; CD safely promotes tested artifacts through deployments using security controls, health checks, observability and rollback procedures.
