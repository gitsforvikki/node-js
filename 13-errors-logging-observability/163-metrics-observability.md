# Lesson 163 — Metrics and Observability Basics

## 1. What is observability?

**Monitoring** tells you that something is wrong. **Observability** helps you understand **why** it is wrong by examining information produced by the running system.

Imagine your Express API suddenly becomes slow:
- Logs show particular requests failing.
- Metrics show response time increased.
- Traces reveal most time was spent waiting for MongoDB.

## 2. Three pillars

| Signal | Question it answers | Example |
| --- | --- | --- |
| Logs | What happened? | Payment failed for request ID |
| Metrics | How much/how often? | 5xx rate 4% |
| Traces | Where did the request spend time? | API → MongoDB → payment service |

These signals work together; no single tool is sufficient.

## 3. Important API metrics

For HTTP services, monitor:
- **Traffic:** requests per second.
- **Errors:** rate of failed requests, especially 5xx.
- **Latency:** how long requests take (p50, p95, p99).
- **Saturation:** CPU, memory, queue depth, connection pool usage.
- **Availability:** percentage of successful service over time.

Example: If p95 latency is 800 ms, approximately 95% of measured requests complete in 800 ms or less for the measured population/window.

## 4. Counter, gauge, histogram

- **Counter:** increases over time (total requests).
- **Gauge:** rises and falls (current active requests).
- **Histogram:** groups observations into buckets to analyze distributions (request duration).

Use a counter for total requests, gauge for queue size, histogram for latency.

## 5. Real Express 5 metrics with prom-client

Install:

```bash
npm install express prom-client
```

```js
import express from "express";
import client from "prom-client";
import { performance } from "node:perf_hooks";

const app = express();
const register = new client.Registry();
client.collectDefaultMetrics({ register });

const httpRequests = new client.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
  registers: [register],
});

const httpDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2, 5],
  registers: [register],
});

app.use((req, res, next) => {
  const start = performance.now();
  let recorded = false;

  function record() {
    if (recorded) return;
    recorded = true;
    // Use bounded, normalized labels; never use raw req.originalUrl.
    const route = req.route?.path ?? "unmatched";
    const labels = {
      method: req.method,
      route,
      status: String(res.statusCode),
    };
    httpRequests.inc(labels);
    httpDuration.observe(labels, (performance.now() - start) / 1000);
  }

  res.once("finish", record);
  res.once("close", record);
  next();
});

app.get("/api/products", (req, res) => {
  res.json({ products: [] });
});

app.get("/metrics", async (req, res) => {
  res.set("Content-Type", register.contentType);
  res.send(await register.metrics());
});

app.listen(3000);
```

**Production note:** Protect `/metrics` via private networking/authentication; it can reveal internal system details. The simplified route label is appropriate for this single-route demo; with mounted routers use a carefully normalized full route template. An aborted response may need a separate outcome label instead of relying only on `statusCode`.

## 6. How Prometheus and Grafana fit

```text
Express API exposes /metrics
        |
        v
Prometheus periodically scrapes metrics
        |
        v
Grafana displays charts / dashboards
        |
        v
Alerting notifies team when thresholds are crossed
```

**Prometheus** stores and queries time-series metrics. **Grafana** visualizes data. They are separate tools, often used together.

## 7. Example PromQL queries

Requests per second:

```promql
sum(rate(http_requests_total[5m]))
```

Approximate 5xx requests per second:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
```

p95 request duration:

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

For a **5xx percentage**, divide the 5xx rate by the total rate and multiply by 100, handling zero-traffic periods appropriately.

## 8. Why metric label cardinality matters

**Bad:**

```text
route="/users/123"
route="/users/456"
route="/users/789"
```

Every unique user ID can create a new time series. This increases memory and storage costs.

**Good:**

```text
route="/users/:id"
```

Never put user emails, tokens, request IDs, or raw URLs in metric labels. Request IDs belong in **logs/traces**, not high-cardinality metrics.

## 9. Distributed tracing with OpenTelemetry

For a request crossing multiple services:

```text
Browser -> API gateway -> Orders API -> Database
                     \-> Payment API
```

A distributed trace contains **spans** representing timed operations. OpenTelemetry provides standard instrumentation and context propagation (such as W3C `traceparent`). Traces can be exported to a collector/backend.

You do not need full tracing for your first Express API; start with structured request logs and basic metrics, then add traces when debugging across services becomes difficult.

## 10. SLI, SLO, and alerting

- **SLI:** measured indicator (e.g., percentage of successful requests).
- **SLO:** reliability target (e.g., 99.9% successful requests over 30 days).
- **SLA:** a contractual service commitment, often with consequences.

Useful alerts: sustained high 5xx rate, abnormal p95 latency, service unavailable, queue backlog, and repeated dependency failures. Avoid paging for a single ordinary 404.

## 11. Common mistakes

1. Collecting only CPU metrics and ignoring user-visible latency.
2. Using raw user IDs/URLs as metric labels.
3. Exposing `/metrics` publicly.
4. Treating logs and metrics as interchangeable.
5. Alerting on every small transient failure.
6. Measuring average latency only and missing slow tail requests.
7. Forgetting multi-process metrics aggregation when running several workers.

## 12. Practice and interview

1. Add `prom-client` and inspect `/metrics`.
2. Call `/api/products` repeatedly and observe the counter.
3. Explain p50 vs p95 vs p99.
4. Design a dashboard for request rate, 5xx rate, and latency.
5. Explain why `requestId` is good in logs but bad as a Prometheus label.

**Interview answer:** Observability combines logs, metrics, and traces to understand production behavior. In Node.js, structured logs, Prometheus-compatible counters/histograms, health checks, and distributed tracing help diagnose errors and latency. Use normalized labels, protect telemetry endpoints, and alert on meaningful user impact.
