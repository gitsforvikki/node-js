# Lesson 20 — npm Scripts

## What are npm Scripts?

npm scripts are named commands stored inside `package.json`.

Example:

```json
{
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js",
    "test": "vitest run"
  }
}
```

Run:

```bash
npm run dev
```

## Why Scripts Matter

They provide one consistent command interface for the whole team.

Instead of remembering:

```bash
NODE_ENV=development node --watch ./src/server.js
```

developers can use:

```bash
npm run dev
```

## Common Scripts

```json
{
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js",
    "test": "vitest run",
    "lint": "eslint .",
    "format": "prettier --write .",
    "build": "tsc",
    "check": "npm run lint && npm run test"
  }
}
```

## Local Binary Resolution

When an npm script runs:

```json
{
  "scripts": {
    "lint": "eslint ."
  }
}
```

npm automatically makes local package binaries available.

You do not need:

```bash
./node_modules/.bin/eslint
```

## Pre and Post Hooks

If you define:

```json
{
  "scripts": {
    "pretest": "npm run lint",
    "test": "vitest run",
    "posttest": "echo tests finished"
  }
}
```

npm can execute lifecycle-related scripts around the main script.

Use this carefully so scripts remain understandable.

## Passing Arguments

```bash
npm test -- --watch
```

Arguments after `--` are passed to the underlying command.

## CI/CD Usage

Jenkins or GitHub Actions might run:

```bash
npm ci
npm run lint
npm test
npm run build
```

This means npm scripts become part of your deployment contract.

## Good Script Design

Prefer:

```json
{
  "scripts": {
    "dev": "...",
    "test": "...",
    "lint": "...",
    "build": "..."
  }
}
```

Avoid obscure names like:

```text
run1
runfinal
abc
temp
```

## Security Note

Remember that scripts are executable commands.

Before installing untrusted packages or running project scripts, understand what they execute.

## Interview Question

### Why use npm scripts instead of running commands directly?

They standardize commands across developers, CI environments, and deployment systems while automatically resolving local package binaries.

## Summary

```text
npm scripts
  = repeatable project commands
  + local binary access
  + CI/CD integration
  + team consistency
```
