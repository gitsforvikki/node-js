# Lesson 14 — Module Resolution

## What is Module Resolution?

Module resolution is the process Node.js uses to answer:

> "When I import this path, which actual file or package should be loaded?"

Example:

```js
import express from "express";
```

Node must find which package `express` refers to.

## Types of Imports

### Relative

```js
import userService from "./user.service.js";
```

### Parent

```js
import config from "../config.js";
```

### Package

```js
import express from "express";
```

### Built-in Node Module

```js
import fs from "node:fs";
```

Using the `node:` prefix makes the intention explicit.

## Package Resolution

For:

```js
import express from "express";
```

Node resolves the package using package metadata and `node_modules` lookup rules.

Conceptually:

```text
current directory
   |
node_modules?
   |
parent directory
   |
node_modules?
   |
continue upward
```

## Why node_modules Can Be Nested

Different packages can depend on different versions.

```text
app
├── node_modules
│   ├── package-a
│   │   └── node_modules
│   │       └── dependency-x
│   └── dependency-x
```

The package manager tries to optimize installation layout, but Node still needs predictable resolution rules.

## package.json and Resolution

Fields such as these influence package entry points:

```json
{
  "main": "./dist/index.js",
  "exports": {
    ".": "./dist/index.js"
  }
}
```

Modern packages often use `exports` to explicitly control what consumers can import.

## Package Exports

Without proper export restrictions, consumers may depend on internal files:

```js
import hiddenThing from "some-package/internal/private.js";
```

This creates fragile coupling.

Using package exports allows authors to define the public API.

## Aliases and Tooling

Frameworks or bundlers may support aliases:

```js
import config from "@/config";
```

But remember:

> Not every alias is a native Node.js feature.

Some aliases come from:
- TypeScript
- bundlers
- framework config
- package import maps / package.json imports

## Common Errors

### MODULE_NOT_FOUND

Usually caused by:
- wrong path
- dependency not installed
- wrong extension
- incorrect package export path
- wrong working directory assumption

### ERR_PACKAGE_PATH_NOT_EXPORTED

Means the package intentionally does not expose that internal path.

## Debugging Approach

Ask:

```text
1. Is this a Node built-in?
2. Is the path relative or package-based?
3. Does the file exist?
4. Is the extension correct?
5. Does package.json expose it?
6. Is the dependency installed?
7. Is ESM/CommonJS compatibility involved?
```

## Interview Question

### How does Node resolve a package import?

Node determines whether the request is built-in, relative, absolute, or package-based, then applies the relevant resolution rules using package metadata and filesystem lookup.

## Summary

Module resolution is the bridge between:

```text
import statement
      |
      v
actual file/package loaded by Node.js
```

Understanding it helps solve many mysterious import errors quickly.
