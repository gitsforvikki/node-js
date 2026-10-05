# Lesson 2 — Node.js vs JavaScript in the Browser

## Core Idea

The JavaScript language is the same, but the **runtime environment is different**.

```text
                JavaScript Language
                 /               \
                /                 \
         Browser Runtime        Node.js Runtime
         DOM                    File System
         window                 process
         document               Buffer
         Web APIs               OS APIs
         localStorage           Streams
```

## Browser JavaScript

A browser provides APIs such as:

```js
window
document
localStorage
navigator
fetch
```

These are not part of the JavaScript language itself.

For example:

```js
document.querySelector("#app");
```

This works because the browser provides the DOM.

## Node.js JavaScript

Node.js provides APIs such as:

```js
process
Buffer
fs
path
http
os
stream
```

Example:

```js
console.log(process.version);
console.log(process.platform);
```

The browser normally does not expose `process`.

## Important Distinction

ECMAScript defines language features:

- variables
- functions
- objects
- classes
- promises
- async/await
- arrays
- modules

The runtime provides environment-specific APIs.

## Example Comparison

### Browser

```js
console.log(window.location.href);
```

### Node.js

```js
console.log(process.cwd());
```

Both are JavaScript, but each runtime provides different capabilities.

## Global Object

Browser:

```js
window
```

Node.js historically used:

```js
global
```

Modern JavaScript also provides:

```js
globalThis
```

which works across environments.

## Security Difference

Browser JavaScript runs inside a heavily restricted sandbox.

If websites could freely access your file system:

```text
malicious-site.com
       |
       v
/home/user/passwords.txt
```

that would be disastrous.

Node.js applications, however, are intentionally allowed to interact with the operating system based on process permissions.

## Module Differences

Historically:

Browser:
```js
import something from "./file.js";
```

Node.js:
```js
const something = require("./file");
```

Modern Node.js supports ES Modules too.

## Event Loop Difference

Both browsers and Node.js have event loops, but their runtime internals are not identical.

Browser event loop integrates:
- rendering
- DOM events
- Web APIs
- user interaction

Node.js integrates:
- network I/O
- file I/O
- sockets
- libuv
- timers
- process lifecycle

## Industry Takeaway

Do not confuse **JavaScript** with the APIs provided by the environment.

```text
JavaScript = Language
Browser    = Runtime Environment
Node.js    = Runtime Environment
```

## Interview Questions

### Can Node.js access the DOM?
Not by default, because Node.js does not provide browser DOM APIs.

### Is `fetch` a JavaScript language feature?
No. It is a runtime API. Modern Node.js also provides a global `fetch`, but that does not make it part of ECMAScript itself.

### Why does `window` not exist in Node.js?
Because `window` represents the browser's global window object, and Node.js does not run inside a browser window.

## Interview-Ready Summary

The language is JavaScript in both environments, but the runtime determines what APIs are available.

Browser JavaScript is optimized for user interfaces and web pages.

Node.js is optimized for operating-system access, servers, networking, scripts, and backend development.
