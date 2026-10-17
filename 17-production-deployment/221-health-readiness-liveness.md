# Lesson 221 — Health, Readiness & Liveness Checks

## 1. What is the difference?
Infrastructure asks three different questions:

| Check | Question | Use |
| --- | --- | --- |
| Liveness | Is this process functioning? | Restart genuinely stuck process |
| Readiness | Should this instance receive new traffic? | Route only to ready instances |
| Startup | Has initialization completed? | Allow slow startup without early restarts |

When MongoDB disconnects, the API may become **not ready** while remaining **alive**. Restarting all API instances during a DB outage may worsen recovery.

## 2. Flow during deployment
```text
New Pod begins -> startup 503 -> DB connects -> startup 200
                 -> readiness 200 -> receives traffic
SIGTERM old Pod -> readiness 503 -> drains -> closes -> exits
```

## 3. Express 5 endpoints
```js
import express from "express";
import mongoose from "mongoose";
const app = express();

let initialized = false;
let shuttingDown = false;

app.get("/health/live", (req, res) =>
  res.status(200).json({ status: "alive" })
);

app.get("/health/startup", (req, res) =>
  res.status(initialized ? 200 : 503)
    .json({ status: initialized ? "started" : "starting" })
);

app.get("/health/ready", (req, res) => {
  const ready = initialized && !shuttingDown &&
    mongoose.connection.readyState === 1;
  res.status(ready ? 200 : 503)
    .json({ status: ready ? "ready" : "not_ready" });
});

async function start() {
  const server = app.listen(3000);
  try {
    await mongoose.connect(process.env.MONGODB_URI);
    initialized = true;
  } catch (err) {
    console.error("Startup failed", err.message);
    await new Promise(resolve => server.close(resolve));
    process.exitCode = 1;
  }
}
start();

// On shutdown: set shuttingDown = true BEFORE draining.
```

This example starts HTTP early so health probes can respond while initialization is ongoing. A complete app must implement the shutdown signal handler and prevent business routes running before initialization.

## 4. Kubernetes configuration
```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 3000
  periodSeconds: 5
  failureThreshold: 30

readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  periodSeconds: 5
  failureThreshold: 2

livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  periodSeconds: 10
  failureThreshold: 3
```
Place these under the application's container in the Pod/Deployment. Tune thresholds/timeouts for startup and infrastructure.

## 5. What should readiness test?
A Mongoose connected state is cheap but might miss a broken DB query. A bounded DB ping can provide stronger evidence at extra cost. Avoid doing expensive dependency calls on every liveness request. Prefer small, reliable checks and monitor the full service separately.

## 6. Security
Never return credentials, URLs containing secrets, stack traces or internal infrastructure details from publicly accessible health endpoints.

## 7. Differences from Lesson 162
Lesson 162 introduced health checks as part of observability. This lesson focuses on **deployment wiring**, Kubernetes probes, startup transitions and how probes cooperate with graceful rollout/shutdown.

## 8. Common mistakes
Returning readiness 200 during drain; treating database outage as liveness failure; using an extremely expensive ready probe; omitting startup window; exposing secret configuration.

## Practice
Implement the three endpoints, simulate DB outage and shutdown, and check statuses. Explain what Kubernetes will do when readiness fails but liveness passes.

**Interview answer:** Liveness detects malfunctioning processes, readiness controls traffic eligibility, and startup probes allow initialization. Use them together to support reliable rolling deployments without cascading restarts.
