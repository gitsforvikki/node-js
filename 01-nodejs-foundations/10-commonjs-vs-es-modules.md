# Lesson 10 — CommonJS vs ES Modules

## Why Module Systems Exist

As applications grow, putting everything in one file becomes impossible to maintain.

Modules allow us to split code into reusable units.

```text
Application
├── auth
├── users
├── payments
├── database
└── utilities
```

Node.js supports two major module systems:

1. CommonJS
2. ES Modules

## CommonJS

Export:

```js
// math.js
function add(a, b) {
  return a + b;
}

module.exports = { add };
```

Import:

```js
const { add } = require("./math");

console.log(add(2, 3));
```

## ES Modules

Export:

```js
// math.js
export function add(a, b) {
  return a + b;
}
```

Import:

```js
import { add } from "./math.js";

console.log(add(2, 3));
```

## package.json Configuration

A common ESM setup:

```json
{
  "type": "module"
}
```

Then `.js` files are treated according to ES Module semantics within that package scope.

Other extensions can explicitly communicate intent:
- `.mjs`
- `.cjs`

## Key Differences

| Area | CommonJS | ES Modules |
|---|---|---|
| Import | `require()` | `import` |
| Export | `module.exports` | `export` |
| Standard | Node-specific historical system | JavaScript standard |
| Loading model | traditionally synchronous | designed for static module structure |
| Top-level await | not standard CJS behavior | supported in ESM |
| Browser compatibility | not native browser module syntax | native standard syntax |

## Static Analysis Advantage

ES Modules use statically analyzable syntax:

```js
import { add } from "./math.js";
```

Tools can understand imports before code executes.

This helps:
- bundlers
- tree shaking
- dependency analysis
- tooling

CommonJS can dynamically require modules:

```js
const moduleName = "./math";
const math = require(moduleName);
```

This flexibility is useful but harder for static tooling.

## Default vs Named Exports

Named:

```js
export const add = () => {};
export const subtract = () => {};
```

Import:

```js
import { add, subtract } from "./math.js";
```

Default:

```js
export default function calculate() {}
```

Import:

```js
import calculate from "./calculate.js";
```

## Modern Recommendation

For new Node.js projects, ES Modules are often a good default when your ecosystem and tooling support them.

However, CommonJS remains extremely important because:
- many existing Node.js codebases use it
- some packages still expose CommonJS
- interviews frequently ask about both
- interoperability issues can appear

A professional Node.js developer should understand both.

## Common Interoperability Problem

You may encounter errors when:
- requiring an ESM-only package from CommonJS
- importing CommonJS from ESM with incorrect assumptions
- mixing extension rules
- forgetting `"type": "module"`

Always check package module format and Node version.

## Industry Architecture Example

```text
src/
├── server.js
├── routes/
├── controllers/
├── services/
├── repositories/
└── utils/
```

Each folder exposes modules that can be imported where needed.

This is the beginning of scalable code organization.

## Interview Questions

### What is CommonJS?
Node.js's historical module system based primarily on `require()` and `module.exports`.

### What are ES Modules?
The standardized JavaScript module system using `import` and `export`.

### Which should you use?
For modern greenfield applications, ESM is often preferred when the stack supports it, but the correct choice depends on ecosystem compatibility, project conventions, runtime version, and dependencies.

### Why are ES Modules easier for static analysis?
Because import/export declarations have a statically defined syntax and module graph that tooling can inspect before executing the module.

## Interview-Ready Summary

```text
CommonJS
  require()
  module.exports

ESM
  import
  export
  standardized JavaScript module system
```

Know both. Prefer consistency inside a project, and avoid mixing module systems without understanding interoperability rules.

## Section 1 Final Mental Model

After these 10 lessons, you should see Node.js like this:

```text
JavaScript
   |
   v
V8 Engine
   |
   v
Node.js Runtime
   |
   +--> Core APIs
   +--> process
   +--> modules
   +--> filesystem/network access
   |
   v
libuv + Operating System
   |
   v
Asynchronous, event-driven backend applications
```

That mental model is the foundation for the next major topics: modules, async programming, event loop internals, streams, HTTP, Express, performance, and production architecture.
