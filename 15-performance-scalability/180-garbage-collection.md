# Lesson 180 — Garbage Collection (GC)

## 1. What is garbage collection?
JavaScript creates objects in memory. **Garbage collection** is V8's process of finding objects that are **no longer reachable** and reclaiming their memory.

You don't usually call `free()` yourself in JavaScript.

## 2. Reachability: simple example
```js
let user = { name: "Vikash", cart: [1, 2, 3] };
user = null; // original object may now become unreachable
```
If no other reference points to the object, it is **eligible** for GC. Memory is not necessarily released immediately; V8 decides when to collect.

```text
GC roots (global variables, active stack, closures, etc.)
    -> reachable objects (keep)
    -> unreachable objects (can reclaim)
```

## 3. Generational GC concept
V8 generally groups objects into:
- **Young generation:** many short-lived objects (e.g., objects created for one API request).
- **Old generation:** objects that live longer.

Young-generation collections are typically more frequent; older collections may be more expensive. V8 uses sophisticated collectors, including incremental/concurrent work, so the exact phases and pauses vary by version.

## 4. Stop-the-world pauses
GC sometimes pauses JavaScript execution. Short pauses may be insignificant; frequent or long pauses can increase API tail latency (p95/p99).

Not all high latency is GC—database wait and CPU loops are other causes. Use profiling and metrics rather than assumptions.

## 5. Observe GC activity
```bash
node --trace-gc src/server.js
```
This prints V8 GC events for **diagnostics**, and can generate lots of output. Run in a controlled environment and look for collection frequency/pause patterns.

For advanced timing:
```js
import { PerformanceObserver, constants } from "node:perf_hooks";
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log("GC duration ms:", entry.duration.toFixed(2));
  }
});
observer.observe({ entryTypes: ["gc"] });
```
This instrumentation has overhead and is typically used during diagnostics; specific metadata and availability depend on Node version.

## 6. What GC cannot solve
```js
const forever = [];
setInterval(() => {
  forever.push({ createdAt: Date.now() });
}, 10);
```
The array continues to reference every object. To GC these objects, remove the references or bound the collection. **A reachable object is not garbage**, even if the app no longer needs it.

## 7. Ways to reduce GC pressure
- Avoid creating huge temporary arrays needlessly.
- Stream or paginate large datasets.
- Reuse appropriately scoped objects when reasonable.
- Bound caches and batch sizes.
- Remove unused event listeners, timers and references.
- Measure before micro-optimizing allocations.

## 8. GC and memory metrics
After GC, `heapUsed` often decreases, but `rss` might not fall immediately. The V8 allocator can retain memory for reuse, and buffers/native allocations behave differently.

A sawtooth heap chart (rising allocations, then falling after GC) is often normal. A continually rising **post-GC baseline** deserves investigation (Lesson 181).

## 9. Common mistakes
Assuming every object is immediately freed after a function returns; thinking GC cleans an unlimited global Map; mistaking any RSS growth for a leak; forcing GC as a regular optimization; assuming GC pauses explain every latency spike.

## Practice
Run `node --trace-gc` on a script that repeatedly creates temporary arrays. Compare with a script retaining those arrays in a global list.

**Interview answer:** V8 automatically reclaims unreachable objects using generational GC. Reachability, not usefulness, determines whether data can be collected. Retained references cause leaks; allocation and GC pauses can affect latency.
