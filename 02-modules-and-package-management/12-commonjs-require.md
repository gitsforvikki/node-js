# Lesson 12 — require() and CommonJS

## What is CommonJS?

CommonJS is Node.js's historical module system.

Its main APIs are:

```js
require()
module.exports
exports
```

## Exporting

```js
function add(a, b) {
  return a + b;
}

module.exports = {
  add,
};
```

Importing:

```js
const { add } = require("./math");
```

## exports vs module.exports

Node exposes:

```js
module.exports
```

and also gives:

```js
exports
```

as a convenience reference.

This works:

```js
exports.add = add;
```

But this can break:

```js
exports = {
  add,
};
```

Why?

Because reassigning `exports` changes only the local variable, not `module.exports`.

Correct:

```js
module.exports = {
  add,
};
```

## require() Resolution

When you write:

```js
require("./math")
```

Node tries to resolve the requested module.

It may look for:
- exact file
- matching extension
- directory entry point
- package metadata

## Module Caching

A major CommonJS concept is caching.

```js
const a = require("./config");
const b = require("./config");

console.log(a === b);
```

Usually:

```text
true
```

The module is loaded once and then reused from cache.

## Why Caching Matters

Suppose:

```js
console.log("config loaded");

module.exports = {
  env: "development",
};
```

If required from five files, that top-level code normally executes once per resolved module identity.

## Singleton-Like Behavior

This makes CommonJS useful for shared instances:

```js
// db.js
const db = createDatabaseConnection();

module.exports = db;
```

Every consumer gets the same cached export.

But be careful with mutable shared state.

## require.cache

Node exposes the internal cache:

```js
console.log(require.cache);
```

You usually should not manipulate it in production code without a strong reason.

## Circular Dependency Example

```text
a.js --> b.js
 ^       |
 |-------|
```

Circular dependencies can result in partially initialized exports.

That is one reason clean dependency direction matters.

## CommonJS Wrapper

Node internally wraps CommonJS modules in a function-like structure.

Conceptually:

```js
(function (exports, require, module, __filename, __dirname) {
  // your module code
});
```

This explains why these values are available inside CommonJS modules.

## Interview Question

### Is require() synchronous?

CommonJS `require()` is traditionally synchronous for module loading.

### Does Node reload the same CommonJS module every time?

Usually no. Node caches the loaded module after the first successful load.

## Best Practices

- Avoid circular dependencies
- Do not mutate shared exported objects carelessly
- Use `module.exports` when replacing the full export object
- Keep module initialization lightweight
- Avoid expensive side effects at import time

## Summary

```text
CommonJS
  require()
  module.exports
  synchronous loading
  module cache
  historical Node.js standard
```
