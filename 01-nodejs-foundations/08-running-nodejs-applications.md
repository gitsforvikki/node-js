# Lesson 8 — Running Node.js Applications

## Basic Execution

Create:

```js
// app.js
console.log("Hello Node.js");
```

Run:

```bash
node app.js
```

## What Happens Internally?

```text
Terminal
   |
   v
node app.js
   |
   v
Operating System starts Node process
   |
   v
Node initializes runtime
   |
   v
V8 loads JavaScript
   |
   v
Code executes
   |
   v
Process exits when no work remains
```

## Node Process Lifecycle

Consider:

```js
console.log("start");
```

The process exits immediately after execution.

But:

```js
setInterval(() => {
  console.log("running");
}, 1000);
```

keeps the event loop active.

## Passing CLI Arguments

Run:

```bash
node app.js Vikash 25
```

Read arguments:

```js
console.log(process.argv);
```

Typical structure:

```text
[
  "/usr/bin/node",
  "/project/app.js",
  "Vikash",
  "25"
]
```

## Environment Variables

Run:

```bash
PORT=5000 node app.js
```

Access:

```js
console.log(process.env.PORT);
```

This is fundamental in production deployments.

## npm Scripts

Instead of repeatedly typing long commands, use:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  }
}
```

Run:

```bash
npm run dev
```

## Watch Mode

Modern Node.js supports built-in watch mode:

```bash
node --watch app.js
```

It restarts when relevant files change.

## Process Exit Codes

A process normally exits with:

```text
0 = success
non-zero = error/failure
```

Example:

```js
process.exitCode = 1;
```

This matters in:
- CI/CD
- Docker
- shell scripts
- Kubernetes
- process managers

## Production Note

Avoid using:

```js
process.exit(1);
```

carelessly in normal request handling because it terminates the entire process.

Production applications usually perform graceful shutdown logic first.

## Interview Question

### When does a Node.js process exit?

When there is no more work keeping the event loop alive, or when it is explicitly terminated.

## Summary

Running Node.js is not just `node app.js`.

A professional developer should understand:
- CLI execution
- arguments
- environment variables
- npm scripts
- process lifecycle
- exit codes
- watch mode
