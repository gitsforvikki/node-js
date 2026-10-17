# Lesson 176 — Measuring Node.js Performance

## 1. Why measure first?
Performance means how quickly and reliably your API handles work with available resources. An app that feels fast for one developer might fail when 500 users arrive together. **Measure before optimizing**: guesses often solve the wrong bottleneck.

Example: `GET /products` may be slow because of a missing MongoDB index, not because Express is slow.

## 2. Important measurements
| Metric | Simple meaning | Example |
| --- | --- | --- |
| Latency | Time for one request | 120 ms |
| Throughput | Requests finished per second (RPS) | 500 RPS |
| Concurrency | Requests in progress together | 40 active |
| p50 / p95 / p99 | 50%, 95%, 99% of requests finish within this time | p95 = 800 ms |
| Error rate | Fraction of requests failing | 2% 5xx |
| CPU / RSS memory | Resource used by the process | 75% CPU |
| Event-loop delay | How long callbacks wait for the loop | p99 100 ms |

**Trade-off:** Throughput can rise while latency grows. Report traffic, errors and percentiles **together**. An average can hide slow users.

## 3. Measure one function with a monotonic clock
```js
import { performance } from "node:perf_hooks";
const start = performance.now();
await User.find({ active: true }).limit(20).lean();
console.log(`DB query took ${(performance.now() - start).toFixed(1)} ms`);
```
`performance.now()` measures elapsed time; it is better than comparing `Date.now()` for short intervals. For meaningful experiments, warm up the app, repeat the measurement, and avoid comparing one lucky run.

## 4. Express timing middleware
```js
import { performance } from "node:perf_hooks";

app.use((req, res, next) => {
  const started = performance.now();
  res.once("finish", () => {
    console.log(JSON.stringify({
      route: req.route?.path ?? "unknown",
      method: req.method,
      status: res.statusCode,
      durationMs: +(performance.now() - started).toFixed(1),
    }));
  });
  next();
});
```
It measures server-side request duration until the response is finished locally; it does **not** include all browser/network delay. The route label can be unavailable or incomplete for mounted routers; use normalized route templates in production.

## 5. Load test your OWN API with autocannon
```bash
npm install -D autocannon
node src/server.js
npx autocannon -c 25 -d 15 http://127.0.0.1:3000/api/products
```
- `-c 25`: 25 concurrent connections.
- `-d 15`: run for 15 seconds.
Read latency percentiles, requests/sec and error counts. Only load test systems you own or are authorized to test. Test representative payloads and auth flows.

## 6. Observe event-loop responsiveness
```js
import { monitorEventLoopDelay } from "node:perf_hooks";

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();
setInterval(() => {
  console.log("event-loop delay p99 ms:", +(histogram.percentile(99) / 1e6).toFixed(1));
  histogram.reset();
}, 10_000).unref();
```
The histogram reports nanoseconds; dividing by 1e6 converts to milliseconds. Delays reveal when JavaScript cannot respond promptly.

## 7. A useful experiment
1. Record baseline p95 and RPS for `GET /products`.
2. Check database query plan/indexes.
3. Change **one thing** (add index, reduce payload, or cache result).
4. Repeat with equal conditions.
5. Keep the change only if it improves the target without increasing errors or memory.

## 8. Mistakes
Optimizing before measuring; reporting only averages; load testing production accidentally; excluding error counts; testing just one user; conflating browser latency with server time.

## Practice / interview
Run a 15-second local load test. Compare p50/p95 before and after deliberately adding a 200 ms delay to one route. Explain how throughput and latency can change together.

**Interview answer:** Start with real workload baselines (latency percentiles, RPS, errors, CPU, memory and event-loop delay). Profile the slow path and change one bottleneck at a time, verifying the result.
