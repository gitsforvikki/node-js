# Lesson 4 — V8 JavaScript Engine

## What is V8?

V8 is Google's high-performance JavaScript engine.

It is used by:
- Chrome
- Node.js
- other Chromium-based environments

Node.js uses V8 to execute JavaScript, but Node.js itself provides much more than V8.

```text
Node.js
├── V8
├── libuv
├── Node core modules
├── bindings
└── runtime APIs
```

## What V8 Does

V8 is responsible for tasks such as:
- parsing JavaScript
- compiling JavaScript
- executing machine code
- memory allocation
- garbage collection
- optimization

## Simplified Execution Flow

```text
JavaScript Source
      |
      v
    Parser
      |
      v
      AST
      |
      v
Interpreter
      |
      v
Bytecode
      |
      v
Optimization
      |
      v
Machine Code
```

## Just-In-Time Compilation

JavaScript is not simply interpreted line-by-line forever.

Modern V8 uses JIT compilation.

Frequently executed code may be optimized based on runtime information.

## Why Stable Object Shapes Matter

Consider:

```js
function createUser(name, age) {
  return { name, age };
}
```

Repeated objects with predictable shapes are easier for V8 to optimize.

Constantly changing object structures may reduce optimization opportunities.

## Example

Less predictable:

```js
const user = {};
user.name = "Vikash";

if (Math.random() > 0.5) {
  user.age = 25;
}
```

More predictable:

```js
const user = {
  name: "Vikash",
  age: null,
};
```

You should not obsess over micro-optimization, but understanding V8 explains why JavaScript performance can change based on code patterns.

## Garbage Collection

V8 automatically manages memory.

Example:

```js
function createObject() {
  const data = { value: 100 };
}

createObject();
```

Once the object is unreachable, it becomes eligible for garbage collection.

Memory leaks still happen when references remain reachable accidentally.

## V8 vs Node.js

V8 can execute:

```js
const a = 10;
const b = 20;
console.log(a + b);
```

But APIs like:

```js
fs.readFile()
process.env
Buffer.from()
```

are Node.js features, not V8 features.

## Interview Question

### Is Node.js built only on V8?

No. V8 executes JavaScript, while Node.js also includes libuv, native bindings, core libraries, networking APIs, file system APIs, streams, process APIs, and more.

## Interview-Ready Summary

```text
V8 = JavaScript execution engine
Node.js = runtime built around V8 + system capabilities
```

V8 handles execution and memory management. Node.js connects JavaScript with the operating system and asynchronous I/O.
