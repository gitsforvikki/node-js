# Lesson 17 — Dependencies vs DevDependencies

## Core Difference

`dependencies` are packages needed for the application to run.

`devDependencies` are packages mainly needed during development, testing, linting, or building.

## Example

```json
{
  "dependencies": {
    "express": "^5.0.0",
    "pg": "^8.0.0"
  },
  "devDependencies": {
    "eslint": "^9.0.0",
    "vitest": "^3.0.0"
  }
}
```

## dependencies

Examples:
- Express
- database drivers
- authentication libraries
- validation libraries
- Redis clients

If the app needs the package at runtime, it generally belongs here.

## devDependencies

Examples:
- ESLint
- Prettier
- test runners
- TypeScript compiler
- development-only tooling

## Install Commands

Production dependency:

```bash
npm install express
```

Development dependency:

```bash
npm install -D eslint
```

## Why This Distinction Matters

It affects:
- deployment image size
- production security surface
- build behavior
- installation strategy
- dependency clarity

## Important Nuance

A package may be a devDependency even if it is required to produce the production build.

Example:

```text
TypeScript compiler
   |
   v
builds JavaScript
   |
   v
production runtime executes built JS
```

The compiler itself may not be needed at runtime.

## Docker Example

Build stage:

```text
install all dependencies
compile application
```

Runtime stage:

```text
copy built output
install only production dependencies
run application
```

This is one reason the distinction matters in multi-stage builds.

## Common Mistakes

### Mistake 1
Putting every package under dependencies.

Result:
- larger production install
- unnecessary attack surface

### Mistake 2
Putting a runtime dependency under devDependencies.

Result:
- application may fail in production when dev packages are omitted

## Interview Question

### What is the difference between dependencies and devDependencies?

Dependencies are required by the application at runtime, while devDependencies are primarily required for development, testing, linting, or build tooling.

## Summary

```text
dependencies
  -> production runtime

devDependencies
  -> development/build/test tooling
```
