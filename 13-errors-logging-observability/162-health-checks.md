# Lesson 162 — Health Check Endpoints

## 1. What is a health check?

A **health check** is an endpoint used by deployment systems, load balancers, and monitors to decide whether an application is running and able to serve requests.

Imagine your Node.js process is alive but MongoDB is disconnected. A simple “server running” check may return 200, while an endpoint that checks readiness should indicate the instance is **not ready**.

## 2. Three useful health concepts

| Check | Question | Typical endpoint |
| --- | --- | --- |
| Liveness | Is this process alive and responding? | `/health/live` |
| Readiness | Can it handle new traffic? | `/health/ready` |
| Startup | Has initialization completed? | `/health/startup` |

The orchestrator can use **liveness** to restart stuck processes and **readiness** to stop routing new requests. Startup probes help slow-starting apps avoid premature liveness failures.

## 3. Basic Express 5 endpoints

```js
import express from "express";
import mongoose from "mongoose";

const app = express();
let shuttingDown = false;
let startupComplete = false;

app.get("/health/live", (req, res) => {
  res.status(200).json({ status: "alive" });
});

app.get("/health/startup", (req, res) => {
  res.status(startupComplete ? 200 : 503).json({
    status: startupComplete ? "started" : "starting",
  });
});

app.get("/health/ready", (req, res) => {
  const ready = startupComplete &&
    !shuttingDown &&
    mongoose.connection.readyState === 1;

  res.status(ready ? 200 : 503).json({
    status: ready ? "ready" : "not_ready",
  });
});

// During startup: await mongoose.connect(...), then set startupComplete = true.
// During shutdown: set shuttingDown = true before draining connections.
```

This example checks Mongoose's **connection state**, not an actual DB query. It is inexpensive but may not detect all database problems.

## 4. Should liveness check MongoDB?

Usually **no**. If the database temporarily fails, restarting every API instance may worsen the outage. Keep liveness focused on whether the process can respond.

Readiness can depend on critical dependencies such as MongoDB, Redis, or required configuration. Choose checks based on what the service actually needs to serve traffic.

## 5. Readiness check with a bounded dependency probe

For cases where connection state is insufficient, use a lightweight, time-bounded check:

```js
app.get("/health/ready", async (req, res) => {
  if (shuttingDown || !startupComplete) {
    return res.sendStatus(503);
  }

  try {
    await mongoose.connection.db.admin().command(
      { ping: 1 },
      { maxTimeMS: 1000 }
    );
    res.status(200).json({ status: "ready" });
  } catch {
    res.status(503).json({ status: "not_ready" });
  }
});
```

**Note:** A database `maxTimeMS` is not a universal end-to-end network timeout. Driver/socket/server-selection settings and infrastructure-level probe timeouts must also be configured. Frequent probes across hundreds of replicas can add load; often cached/connection-state checks are better.

## 6. What should health responses contain?

Good:

```json
{ "status": "ready" }
```

Avoid exposing:
- database connection strings
- stack traces
- private hostnames and network details
- API keys and environment variables
- detailed internal topology on public endpoints

Expose deeper diagnostics only to authenticated internal monitoring.

## 7. Health checks and graceful shutdown

```text
SIGTERM received
  -> readiness becomes 503
  -> load balancer stops sending new traffic
  -> server drains in-flight requests
  -> DB and queues close
  -> process exits
```

Depending on your deployment, allow a short drain period so readiness changes propagate before closing the listener. See Lesson 157.

## 8. Kubernetes example

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /health/startup
    port: 3000
  periodSeconds: 5
  failureThreshold: 30
```

This is an illustrative portion of a Kubernetes container specification. Tune intervals, thresholds, and timeouts to your application's startup time and infrastructure.

## 9. Testing with curl

```bash
curl -i http://localhost:3000/health/live
curl -i http://localhost:3000/health/ready
```

Test during startup, a temporary DB outage, and shutdown. A correct health design must behave sensibly **when dependencies fail**, not only when everything works.

## 10. Common mistakes

1. Making liveness fail whenever MongoDB disconnects.
2. Returning 200 for readiness while shutting down.
3. Running expensive DB queries on every health probe.
4. Exposing internal secrets in health responses.
5. Using one endpoint for every probe without considering different purposes.
6. Forgetting to set startup state after initialization.

## 11. Practice and interview

1. Implement `/health/live` and `/health/ready`.
2. Simulate a DB disconnection and compare their responses.
3. Explain why liveness and readiness should differ.
4. Add a readiness transition to graceful shutdown.

**Interview answer:** Health endpoints tell infrastructure whether a service is alive, initialized, and ready for traffic. Liveness should not depend unnecessarily on external services; readiness should reflect critical dependencies and shutdown state.
