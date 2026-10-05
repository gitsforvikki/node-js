# Lesson 24 — fs/promises

## Why fs/promises Exists

The callback-based `fs` API works well, but modern Node.js code often uses async/await.

Node provides promise-based filesystem APIs through:

```js
import { readFile, writeFile } from "node:fs/promises";
```

## Reading a File

```js
import { readFile } from "node:fs/promises";

const content = await readFile("data.txt", "utf8");

console.log(content);
```

## Writing

```js
import { writeFile } from "node:fs/promises";

await writeFile(
  "output.txt",
  "Hello from fs/promises"
);
```

## Error Handling

```js
try {
  const content = await readFile("data.txt", "utf8");
  console.log(content);
} catch (error) {
  console.error("Unable to read file:", error.message);
}
```

## Directory Example

```js
import {
  mkdir,
  readdir,
} from "node:fs/promises";

await mkdir("uploads/images", {
  recursive: true,
});

const files = await readdir("uploads");

console.log(files);
```

## Remove Files and Directories

```js
import {
  unlink,
  rm,
} from "node:fs/promises";

await unlink("temp.txt");

await rm("temp-folder", {
  recursive: true,
  force: true,
});
```

Be careful with recursive deletion.

## stat

```js
import { stat } from "node:fs/promises";

const info = await stat("data.txt");

console.log({
  size: info.size,
  isFile: info.isFile(),
});
```

## Promise Parallelism

Suppose you need three independent files:

Sequential:

```js
const a = await readFile("a.txt", "utf8");
const b = await readFile("b.txt", "utf8");
const c = await readFile("c.txt", "utf8");
```

Parallel:

```js
const [a, b, c] = await Promise.all([
  readFile("a.txt", "utf8"),
  readFile("b.txt", "utf8"),
  readFile("c.txt", "utf8"),
]);
```

If operations are independent, parallel execution may reduce total wait time.

## But Do Not Create Unlimited Parallelism

Bad:

```js
await Promise.all(
  millionFiles.map((file) => readFile(file))
);
```

This can overwhelm:
- memory
- filesystem
- file descriptor limits
- thread pool

Production systems often use bounded concurrency.

## File Handles

```js
import { open } from "node:fs/promises";

const file = await open("data.txt", "r");

try {
  const content = await file.readFile("utf8");
  console.log(content);
} finally {
  await file.close();
}
```

Always close manually managed file handles.

## fs vs fs/promises

```text
node:fs
  callback + sync + stream APIs

node:fs/promises
  promise-based async filesystem APIs
```

Streams still come from `node:fs`.

## Industry Recommendation

For ordinary async file operations, `fs/promises` often produces cleaner code.

For large-file streaming, use stream APIs instead.

## Interview Question

### Why prefer fs/promises in modern code?

It integrates naturally with promises and async/await, making asynchronous control flow easier to compose and read.

## Summary

```text
fs/promises
    |
    +--> clean async/await
    +--> try/catch errors
    +--> Promise.all
    +--> no callback nesting
```

## Practice Task

Build an async script that:
- creates a folder
- writes 3 files
- reads all 3 in parallel
- prints their contents
- removes the folder safely
