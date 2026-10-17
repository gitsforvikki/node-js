# Lesson 179 — Memory Management in Node.js

## 1. Where does memory go?
Node.js uses **V8** for JavaScript memory and also allocates memory outside the JS heap for Buffers, native libraries and runtime structures.

```text
Node.js process (RSS)
  +-- V8 JS heap (objects, strings, arrays)
  +-- Buffers / ArrayBuffers / native allocations
  +-- Stack and code/runtime overhead
  +-- Shared libraries and other memory
```
**RSS** (resident set size) is approximate physical memory held by the process; it is not equal to JavaScript heap size.

## 2. Inspect process.memoryUsage()
```js
const used = process.memoryUsage();
for (const [name, bytes] of Object.entries(used)) {
  console.log(name, (bytes / 1024 / 1024).toFixed(1), "MB");
}
```
Common fields:
- `rss`: memory resident for process.
- `heapTotal`: allocated V8 heap capacity.
- `heapUsed`: currently used JS heap.
- `external`: memory tied to native/C++ objects reachable from JS.
- `arrayBuffers`: Buffer/ArrayBuffer allocations (also included in external, not additional to it).

Do **not** sum these fields as if each is a separate piece; some overlap.

## 3. Streaming vs buffering
Unsafe for giant files:
```js
import { readFile } from "node:fs/promises";
const entireFile = await readFile("huge-video.mp4"); // RAM usage grows with file
```
Better for transfer:
```js
import { createReadStream } from "node:fs";
import { pipeline } from "node:stream/promises";
await pipeline(createReadStream("huge-video.mp4"), writableDestination);
```
The destination must be a writable stream. Streaming processes data in chunks and cooperates with backpressure; it avoids retaining the **entire file** in RAM.

## 4. Objects and references
```js
const cache = new Map();
cache.set("user-1", { name: "Vikash" });
```
As long as `cache` strongly references that object, garbage collection cannot reclaim it—even if the request that created it finished. This is the foundation of many memory leaks.

## 5. Limits and configuration
V8's heap limit is not the total process memory limit. A Buffer-heavy process can exceed container RAM while heap usage looks modest.

Example:
```bash
node --max-old-space-size=1024 src/server.js
```
This changes the old-generation V8 heap limit in MB, **not** a total process RAM quota; increasing it can postpone out-of-memory failure while making GC pauses longer. Fix retained data and bound workloads before simply increasing memory.

## 6. Practical memory safety
- Paginate DB queries instead of `find({})` on giant collections.
- Use `.lean()` for read-only Mongoose queries when document methods aren't needed.
- Stream large uploads/downloads rather than buffering.
- Bound caches (TTL/max entries), request sizes and concurrency.
- Close streams/handles and remove listeners when done.
- Measure process RSS and heap trends over time.

## 7. Example: memory-saving pagination
```js
const page = Math.max(1, Number(req.query.page) || 1);
const limit = 50; // fixed bounded limit for this example
const products = await Product.find()
  .sort({ _id: 1 })
  .skip((page - 1) * limit)
  .limit(limit)
  .lean();
res.json(products);
```
For deep pagination, cursor/keyset pagination is often more efficient than large `skip()` offsets. Validate query parameters before using them.

## 8. Common mistakes
Treating heap as total memory; holding unlimited in-memory caches; reading entire uploads into RAM; assuming increased heap size fixes leaks; leaking EventEmitter listeners; using `global.gc()` as routine performance tuning.

## Practice
Print memory usage before/after allocating a 10 MB Buffer; compare `heapUsed`, `external`, and `rss`. Results can vary due to allocation strategy and OS behavior.

**Interview answer:** Node.js uses V8 heap plus native/external memory. Monitor heap and RSS separately, avoid retaining unnecessary references, use bounded data structures and streaming, and investigate sustained memory growth instead of blindly increasing heap limits.
