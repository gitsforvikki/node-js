# Lesson 22 — path

## What you'll learn
- Why path handling matters in Node.js
- How to build cross-platform file paths safely
- Difference between join, resolve, basename, dirname and extname
- Common production mistakes
- Interview-ready concepts

## Core Concept

The `path` module helps you work with file and directory paths in a platform-safe way.

Import it with:

```js
import path from "node:path";
```

or in CommonJS:

```js
const path = require("node:path");
```

## Why Not Build Paths Manually?

Bad:

```js
const file = __dirname + "/uploads/" + filename;
```

This assumes a separator style.

Different operating systems use different path conventions.

Better:

```js
const file = path.join(__dirname, "uploads", filename);
```

## path.join()

Combines path segments and normalizes separators.

```js
const fullPath = path.join("users", "vikash", "profile.json");

console.log(fullPath);
```

Conceptually:

```text
users + vikash + profile.json
          |
          v
users/vikash/profile.json
```

## path.resolve()

Builds an absolute path by resolving from right to left until an absolute segment is found.

```js
const absolute = path.resolve("src", "config", "app.js");

console.log(absolute);
```

If current working directory is:

```text
/home/vikash/project
```

result may be:

```text
/home/vikash/project/src/config/app.js
```

## join vs resolve

```js
path.join("src", "app.js");
path.resolve("src", "app.js");
```

Think:

```text
join
  -> combine path pieces

resolve
  -> produce absolute resolved path
```

## path.basename()

Returns the final part of a path.

```js
path.basename("/home/user/file.txt");
// file.txt
```

Remove extension:

```js
path.basename("/home/user/file.txt", ".txt");
// file
```

## path.dirname()

Returns the directory portion.

```js
path.dirname("/home/user/file.txt");
// /home/user
```

## path.extname()

Returns the file extension.

```js
path.extname("photo.png");
// .png
```

Useful in:
- upload validation
- file classification
- static file logic

Do not treat extension alone as a secure file type check.

## path.parse()

```js
console.log(path.parse("/home/user/file.txt"));
```

Produces a structured object conceptually like:

```js
{
  root: "/",
  dir: "/home/user",
  base: "file.txt",
  ext: ".txt",
  name: "file"
}
```

## path.format()

Rebuilds a path from an object.

```js
path.format({
  dir: "/home/user",
  name: "report",
  ext: ".pdf",
});
```

## path.sep

Platform-specific separator.

```js
console.log(path.sep);
```

On Linux/macOS usually:

```text
/
```

On Windows:

```text
\
```

## path.delimiter

Used in environment variables such as PATH.

Linux/macOS:

```text
:
```

Windows:

```text
;
```

## Production Example — Upload Directory

```js
import path from "node:path";

const uploadDir = path.resolve(
  process.cwd(),
  "storage",
  "uploads"
);
```

Now your application has one stable absolute path.

## Security: Path Traversal

Never blindly combine user input into filesystem paths.

Dangerous input:

```text
../../../../etc/passwd
```

A user may try to escape the intended directory.

Example protection idea:

```js
const root = path.resolve("uploads");
const requested = path.resolve(root, userInput);

if (!requested.startsWith(root + path.sep)) {
  throw new Error("Invalid path");
}
```

Real production code should carefully validate allowed filenames and path boundaries.

## Common Mistakes

- concatenating paths manually
- confusing `process.cwd()` with module location
- trusting file extensions
- accepting raw user-supplied paths
- assuming Linux path syntax everywhere

## Interview Questions

### What is the difference between path.join and path.resolve?

`path.join()` combines segments and normalizes them. `path.resolve()` resolves segments into an absolute path.

### Why use the path module?

For portable, safe, and readable filesystem path manipulation across operating systems.

## Interview-Ready Summary

```text
path.join()     -> combine
path.resolve()  -> absolute path
path.basename() -> file/base name
path.dirname()  -> directory
path.extname()  -> extension
path.parse()    -> path to object
path.format()   -> object to path
```

## Practice Task

Create a script that receives a filename and prints:
- absolute path
- directory
- base name
- extension
- parsed path object
