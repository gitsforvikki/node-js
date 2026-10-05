# Lesson 9 — global, process, __dirname and __filename

## Why Globals Matter

Node.js provides runtime-specific global values that are useful across applications.

Important examples:
- `global`
- `globalThis`
- `process`
- `Buffer`
- timers

CommonJS also exposes useful file-related values:
- `__dirname`
- `__filename`

## global

```js
console.log(global);
```

`global` is Node.js's historical global object.

Modern portable code can use:

```js
globalThis
```

## process

`process` represents the currently running Node.js process.

Useful properties:

```js
console.log(process.pid);
console.log(process.platform);
console.log(process.version);
console.log(process.cwd());
console.log(process.env.NODE_ENV);
```

## process.argv

Command:

```bash
node app.js hello
```

Code:

```js
console.log(process.argv);
```

Useful for CLI tools.

## process.env

Environment configuration:

```js
const port = process.env.PORT ?? 3000;
```

Production applications frequently use environment variables for:
- database URLs
- ports
- API keys
- secrets
- service endpoints
- environment names

Never commit secrets into Git.

## __dirname

In CommonJS:

```js
console.log(__dirname);
```

It gives the directory containing the current module.

## __filename

In CommonJS:

```js
console.log(__filename);
```

It gives the full path of the current file.

## Important ES Module Difference

In ES Modules, `__dirname` and `__filename` are not historically available in the same way as CommonJS.

A portable ESM pattern is based on:

```js
import.meta.url
```

In modern Node.js versions, additional conveniences may exist depending on version, so always know which Node version your project targets.

## process.cwd() vs __dirname

This is frequently misunderstood.

`process.cwd()`:

> Directory from which the Node.js process was started.

`__dirname`:

> Directory of the current CommonJS module file.

Example project:

```text
/project
  /src
    app.js
```

If you run:

```bash
cd /project
node src/app.js
```

then conceptually:

```text
process.cwd()  -> /project
__dirname      -> /project/src
```

## Industry Use

This difference matters for:
- loading configuration
- reading templates
- serving static files
- migrations
- file uploads
- resolving certificates

## Interview Question

### Difference between process.cwd() and __dirname?

`process.cwd()` returns the current working directory from which the process was started, while `__dirname` identifies the directory of the current CommonJS module.

## Summary

```text
global/globalThis -> global runtime object
process           -> current Node process
process.env       -> environment configuration
process.argv      -> CLI arguments
process.cwd()     -> working directory
__dirname         -> current module directory
__filename        -> current module file
```
