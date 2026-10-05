# Lesson 15 — package.json

## What is package.json?

`package.json` is the metadata and configuration file for a Node.js package/project.

It describes:
- project identity
- scripts
- dependencies
- module type
- engines
- package entry points
- publishing metadata

## Minimal Example

```json
{
  "name": "my-api",
  "version": "1.0.0"
}
```

## Common Fields

### name

```json
{
  "name": "careerloop-api"
}
```

Should be valid for the package ecosystem.

### version

```json
{
  "version": "1.2.3"
}
```

Usually follows semantic versioning.

### scripts

```json
{
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js",
    "test": "vitest"
  }
}
```

### dependencies

```json
{
  "dependencies": {
    "express": "^5.0.0"
  }
}
```

### devDependencies

```json
{
  "devDependencies": {
    "eslint": "^9.0.0"
  }
}
```

### type

```json
{
  "type": "module"
}
```

Affects how Node interprets `.js` files.

## engines

```json
{
  "engines": {
    "node": ">=22"
  }
}
```

Useful for documenting supported runtime versions.

## private

```json
{
  "private": true
}
```

Useful for applications that should never be accidentally published to npm.

## main

Historically:

```json
{
  "main": "index.js"
}
```

Defines the primary package entry point for consumers.

## exports

Modern package API control:

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./client": "./dist/client.js"
  }
}
```

This gives package authors better encapsulation.

## package.json Is Not Just Documentation

It directly affects:
- package installation
- runtime behavior
- scripts
- publishing
- module resolution
- CI/CD
- package managers

## Production Example

```json
{
  "name": "node-api",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js",
    "test": "vitest run"
  },
  "engines": {
    "node": ">=22"
  }
}
```

## Common Mistakes

- forgetting `private: true` for private apps
- putting production packages into devDependencies
- using vague scripts
- mixing CommonJS and ESM unintentionally
- editing lockfiles manually

## Interview Question

### Why is package.json important?

Because it defines package metadata, dependencies, scripts, runtime/module behavior, supported engines, and package entry points.

## Interview-Ready Summary

```text
package.json
  = project identity
  + dependency manifest
  + script runner config
  + module config
  + package metadata
```
