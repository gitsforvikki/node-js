# Lesson 11 — Creating and Exporting Modules

## Why Modules Matter

As a Node.js application grows, keeping everything in one file becomes difficult to understand, test, and maintain.

Modules solve this problem by splitting the application into focused, reusable pieces.

```text
Application
├── users
├── auth
├── payments
├── database
└── utilities
```

A good module should ideally have one clear responsibility.

## Basic CommonJS Example

```js
// math.js
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

module.exports = {
  add,
  subtract,
};
```

Use it:

```js
const { add, subtract } = require("./math");

console.log(add(10, 5));
console.log(subtract(10, 5));
```

## Basic ES Module Example

```js
// math.js
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}
```

Use it:

```js
import { add, subtract } from "./math.js";

console.log(add(10, 5));
```

## Named vs Default Exports

### Named Export

```js
export const createUser = () => {};
export const deleteUser = () => {};
```

Import:

```js
import { createUser, deleteUser } from "./user.service.js";
```

### Default Export

```js
export default function createUser() {}
```

Import:

```js
import createUser from "./user.service.js";
```

## Industry Structure

```text
src/
├── modules/
│   ├── user/
│   │   ├── user.controller.js
│   │   ├── user.service.js
│   │   ├── user.repository.js
│   │   └── user.routes.js
│   └── auth/
│       ├── auth.controller.js
│       └── auth.service.js
└── server.js
```

Each file acts as a module with one concern.

## Practical Example

```js
// user.service.js
export function getUserById(id) {
  return {
    id,
    name: "Vikash",
  };
}
```

```js
// user.controller.js
import { getUserById } from "./user.service.js";

export function getUser(req, res) {
  const user = getUserById(req.params.id);
  res.json(user);
}
```

## Best Practices

- Keep modules focused
- Avoid circular dependencies
- Prefer clear file names
- Avoid exporting everything from everywhere
- Keep internal helpers private when they are not needed outside the module
- Prefer feature-based organization for larger applications

## Common Mistake

Do not create one huge utility file:

```text
utils.js
  2500 lines
```

This becomes a dumping ground.

Prefer:

```text
utils/
├── date.js
├── crypto.js
├── validation.js
└── string.js
```

## Interview Question

### What is a module in Node.js?

A module is an isolated unit of code that can expose selected functionality to other parts of the application through exports and imports.

## Interview-Ready Summary

```text
Module
  = isolated file/unit
  + explicit exports
  + reusable functionality
  + maintainable architecture
```

## Practice Task

Create:

```text
calculator/
├── add.js
├── subtract.js
└── index.js
```

Export all functions from `index.js` and consume them from another file.
