# Lesson 13 — import / export and ES Modules

## What are ES Modules?

ES Modules, commonly called ESM, are the standard JavaScript module system.

Main syntax:

```js
import
export
```

## Basic Export

```js
export function add(a, b) {
  return a + b;
}
```

Import:

```js
import { add } from "./math.js";
```

## Default Export

```js
export default class UserService {}
```

Import:

```js
import UserService from "./user.service.js";
```

## Enable ESM in Node.js

A common configuration:

```json
{
  "type": "module"
}
```

Then `.js` files use ESM semantics in that package scope.

Explicit extensions are also available:

```text
.mjs -> ES Module
.cjs -> CommonJS
```

## Static Module Structure

ESM imports are statically analyzable.

```js
import { createUser } from "./user.service.js";
```

The dependency graph can be understood before normal execution.

This enables:
- tree shaking
- better tooling
- dependency graph analysis
- bundler optimization

## Dynamic Import

ESM also supports dynamic loading:

```js
const module = await import("./feature.js");
```

This is useful when code should load conditionally or lazily.

## Top-Level await

ES Modules can use top-level await:

```js
const response = await fetch("https://example.com");
```

Use carefully because module loading can become dependent on async initialization.

## import.meta

ESM provides metadata through:

```js
console.log(import.meta.url);
```

Useful for locating the current module.

## Common Mistake

In ESM:

```js
require("./file");
```

does not behave like native CommonJS usage.

Similarly, `__dirname` and `__filename` are not part of classic ESM semantics.

## Production Advice

Pick one module system for the project unless compatibility requires mixing.

A consistent codebase is easier to:
- debug
- test
- lint
- deploy
- onboard developers into

## CommonJS Interop

You may consume older CommonJS packages from ESM, but export shape can be confusing.

Always inspect package documentation if imports do not behave as expected.

## Interview Questions

### Why are ES Modules considered more standard?

Because `import` and `export` are defined as part of the ECMAScript language standard and supported across multiple JavaScript environments.

### What is the role of "type": "module"?

It tells Node.js to interpret `.js` files in the relevant package scope as ES Modules.

## Interview-Ready Summary

```text
ESM
  import/export
  standard JavaScript modules
  statically analyzable
  top-level await support
  modern Node.js default choice for many new projects
```

## Practice Task

Create three files:
- `user.service.js`
- `user.controller.js`
- `app.js`

Use only ESM syntax to connect them.
