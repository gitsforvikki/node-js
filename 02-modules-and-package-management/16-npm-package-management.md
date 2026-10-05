# Lesson 16 — npm and Package Management

## What is npm?

npm is a package manager and package registry ecosystem widely used with Node.js.

It helps with:
- installing packages
- managing versions
- running scripts
- publishing packages
- dependency resolution

## Initialize a Project

```bash
npm init
```

Fast version:

```bash
npm init -y
```

## Install a Package

```bash
npm install express
```

Short form:

```bash
npm i express
```

## Install a Development Dependency

```bash
npm install -D eslint
```

## Remove a Package

```bash
npm uninstall express
```

## Install All Dependencies

```bash
npm install
```

npm reads:
- `package.json`
- `package-lock.json`

and installs the dependency graph.

## npm ci

For CI environments:

```bash
npm ci
```

This is especially useful because it performs a clean, lockfile-driven installation.

Typical use cases:
- GitHub Actions
- Jenkins
- Docker builds
- production CI pipelines

## npm install vs npm ci

```text
npm install
  more flexible
  may update lockfile

npm ci
  requires lockfile consistency
  clean deterministic install
  ideal for CI
```

## Global Installation

```bash
npm install -g some-cli
```

Use global installs mainly for command-line tools when appropriate.

Project dependencies should usually be local.

## npm exec / npx

Run package binaries without manually referencing `node_modules/.bin`.

Example:

```bash
npx eslint .
```

Modern npm can also use:

```bash
npm exec eslint .
```

## npm audit

```bash
npm audit
```

Checks known dependency vulnerabilities.

Important:

Do not blindly run destructive upgrade commands without reviewing impact.

## npm outdated

```bash
npm outdated
```

Shows packages with newer versions available.

## Production Principle

Package management is not just "install libraries."

It affects:
- security
- reproducibility
- deployment
- supply-chain risk
- build stability

## Common Mistakes

- deleting lockfiles casually
- using global packages instead of local project dependencies
- blindly upgrading major versions
- running `npm audit fix --force` without understanding changes
- committing `node_modules`

## .gitignore

Usually:

```text
node_modules/
.env
```

## Interview Question

### Why use npm ci in CI/CD?

Because it installs dependencies directly from the lockfile using a clean install and fails when package metadata and lockfile are inconsistent, improving reproducibility.

## Summary

```text
npm
  install dependencies
  manage versions
  run scripts
  publish packages
  support reproducible builds
```
