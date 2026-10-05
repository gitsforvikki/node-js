# Lesson 30 — util

## What is the util Module?

`node:util` contains utilities that support common Node.js development patterns.

Import:

```js
import util from "node:util";
```

Some APIs are especially useful for:
- callback interoperability
- debugging
- inspection
- formatting
- type checks

## util.promisify()

Older Node-style APIs often follow:

```js
function callback(err, result) {}
```

`promisify` converts compatible callback APIs into promise-returning functions.

Example:

```js
import { promisify } from "node:util";
import crypto from "node:crypto";

const scryptAsync = promisify(crypto.scrypt);

const key = await scryptAsync(
  "password",
  "salt",
  64
);
```

This is common in modern async/await code.

## When promisify Works

It expects Node-style callbacks:

```text
callback(error, result)
```

It may not work correctly for unusual callback signatures without custom handling.

## util.callbackify()

The reverse idea:

```js
import { callbackify } from "node:util";

async function getUser() {
  return { id: 1 };
}

const getUserCallback =
  callbackify(getUser);
```

Useful mainly when integrating promise-based logic with legacy callback APIs.

## util.inspect()

Useful for detailed object debugging.

```js
import { inspect } from "node:util";

const obj = {
  user: {
    profile: {
      name: "Vikash",
    },
  },
};

console.log(
  inspect(obj, {
    depth: null,
  })
);
```

## console.log and inspect

Node internally uses inspection behavior when displaying objects.

`util.inspect` gives more control over:
- depth
- colors
- hidden properties
- array size
- formatting

## util.format()

```js
import { format } from "node:util";

const message = format(
  "User %s has id %d",
  "Vikash",
  10
);
```

Useful in lower-level logging or tooling, though template literals are often simpler for ordinary code.

## util.types

Node exposes runtime type helpers.

Example:

```js
import { types } from "node:util";

console.log(
  types.isDate(new Date())
);
```

Useful when built-in type detection is needed.

## util.deprecate()

Library authors can mark APIs as deprecated.

```js
import { deprecate } from "node:util";

const oldFunction = deprecate(
  () => {
    console.log("running");
  },
  "oldFunction is deprecated"
);
```

Useful when maintaining reusable packages.

## Industry Example — Promise Wrapper

Imagine a legacy API:

```js
function legacyQuery(sql, callback) {
  // callback(err, result)
}
```

Convert once:

```js
const query = promisify(legacyQuery);
```

Then application code can use:

```js
const result = await query(sql);
```

## Common Mistakes

- promisifying APIs that already return promises
- promisifying methods that depend on `this` without binding
- using inspect output as serialized application data
- relying on formatting utilities for business logic

## this Binding Problem

Suppose:

```js
const method = promisify(object.method);
```

If `method` requires `this`, this may fail.

Bind it:

```js
const method = promisify(
  object.method.bind(object)
);
```

## Interview Questions

### What does util.promisify do?

It converts Node-style error-first callback functions into promise-returning functions.

### What is util.inspect for?

It produces configurable string representations of JavaScript values, mainly for debugging and diagnostics.

## Summary

```text
util
  promisify()
  callbackify()
  inspect()
  format()
  types
  deprecate()
```

Not every API is used daily, but `promisify` and `inspect` are especially practical.

## Practice Task

Take one callback-based function and:
1. promisify it
2. await the result
3. handle its errors with try/catch
