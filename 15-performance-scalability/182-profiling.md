# Lesson 182 — Profiling Node.js Applications

## 1. What is profiling?
**Profiling** measures **where a running application spends time or retains memory**. A benchmark says “the endpoint is slow”; a profile helps answer “which function is slow?”

Example: `GET /products` takes 900 ms. The cause could be inefficient JavaScript, JSON serialization, a slow MongoDB query, or network latency. **CPU profiling primarily reveals CPU work, not time waiting on a remote database.**

## 2. Performance toolbox
| Tool | Best for |
| --- | --- |
| `performance.now()` | Timing a known code section |
| `node --cpu-prof` | CPU-intensive JS/native call stacks |
| Chrome DevTools CPU profile | Exploring call stacks and hot paths |
| Heap snapshot | Understanding retained objects |
| `monitorEventLoopDelay()` | Detecting delayed callbacks |
| Database `explain()` | Query/index problems |
| APM / OpenTelemetry | Tracing latency across services |

## 3. Create and inspect a CPU profile
```bash
node --cpu-prof src/server.js
# In another terminal (against your own local API):
npx autocannon -c 15 -d 15 http://127.0.0.1:3000/api/report
```
Stop the server normally so its CPU profile is written. Node creates a `.cpuprofile` file (location/name depend on configuration). Open Chrome DevTools → Performance / JavaScript Profiler and load the profile, or use tools that accept V8 CPU profiles.

**Hot function:** a function accounting for substantial CPU samples. **Self time** refers to work in that function itself; **total time** includes called functions.

## 4. Reproduce a CPU bottleneck
```js
function expensiveCalculation(n) {
  let total = 0;
  for (let i = 0; i < n; i++) total += Math.sqrt(i);
  return total;
}
app.get("/api/report", (req, res) => {
  res.json({ total: expensiveCalculation(25_000_000) });
});
```
The large loop intentionally blocks the event loop. Compare CPU profiles and p95 latency after replacing it with a precomputed result or a worker (Lesson 187).

## 5. Profile asynchronous applications carefully
CPU profiles show work executing on the CPU. They **do not directly show MongoDB server time** during an `await`. Measure database calls individually:
```js
import { performance } from "node:perf_hooks";

const start = performance.now();
const products = await Product.find({ active: true }).lean();
console.log("Mongo query elapsed ms:", performance.now() - start);
```
Then inspect server-side query plans, indexes, network latency and query volume. An application can have **low CPU and high request latency**.

## 6. Heap snapshots
Start locally with `node --inspect src/server.js` and open DevTools → Memory. Take a baseline snapshot, create repeated work, then take another and compare retained objects.

**Warning:** snapshots can pause execution, temporarily double memory usage and contain private data. Never expose the inspector publicly or capture sensitive production data casually.

## 7. Controlled optimization process
1. Collect realistic load, p50/p95/p99 and error rate.
2. Capture CPU and memory evidence.
3. Identify the actual bottleneck.
4. Make one change (index, algorithm, caching, worker).
5. Repeat identical workload.
6. Compare throughput, latency, CPU, memory and correctness.

## 8. Common mistakes
Profiling only startup instead of workload; assuming CPU profiles show external I/O wait; optimizing functions that are rarely used; exposing debug ports; claiming success based on one run.

## Practice
Profile an intentionally expensive report route. Find the hottest function, change the implementation and rerun the same load test.

**Interview answer:** Profiling identifies CPU hot paths and memory retention under realistic workloads. Combine CPU profiles, heap snapshots, event-loop measurements and database tracing; optimize verified bottlenecks instead of guessing.
