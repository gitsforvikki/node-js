# Lesson 220 — Graceful Deployment

## 1. What is graceful deployment?
When releasing a new version, your app may still be serving HTTP requests, Socket.IO connections and background jobs. **Graceful deployment** brings up healthy replacements before draining and stopping old instances.

```text
New instance starts -> DB connects -> readiness 200
     -> traffic shifts to new instance
     -> old instance readiness 503
     -> in-flight requests finish (bounded)
     -> old HTTP/queue/socket resources close
     -> old process exits
```

## 2. Express 5 + MongoDB example
```js
import express from "express";
import mongoose from "mongoose";

const app = express();
let draining = false;

app.get("/health/live", (req, res) => res.sendStatus(200));
app.get("/health/ready", (req, res) =>
  res.sendStatus(!draining && mongoose.connection.readyState === 1 ? 200 : 503)
);

async function start() {
  await mongoose.connect(process.env.MONGODB_URI);
  const server = app.listen(3000);

  async function shutdown(signal) {
    if (draining) return;
    draining = true;
    console.log("Draining:", signal);
    const hardStop = setTimeout(() => process.exit(1), 20_000);
    hardStop.unref();

    try {
      await new Promise((resolve, reject) => {
        server.close(err => err ? reject(err) : resolve());
      });
      await mongoose.disconnect();
      clearTimeout(hardStop);
      process.exitCode = 0;
    } catch (error) {
      console.error("Shutdown error", error);
      process.exitCode = 1;
    }
  }

  process.once("SIGTERM", () => void shutdown("SIGTERM"));
  process.once("SIGINT", () => void shutdown("SIGINT"));
}
start().catch(err => { console.error(err); process.exitCode = 1; });
```

## 3. Request draining explained
`server.close()` stops accepting new connections and waits for in-flight HTTP requests. A hard timeout prevents endless hangs. On some deployment platforms, allow a short pre-stop drain period for readiness state changes to propagate **before** closing the listener.

## 4. Extra resources
The HTTP server isn't the only resource. Close Socket.IO/WebSocket connections (inform clients to reconnect), stop BullMQ workers and scheduled jobs, release Redis/DB connections, and ensure failed/interrupted work is retried safely.

## 5. Rolling, blue-green and canary
- **Rolling:** gradually replace old replicas with new replicas.
- **Blue-green:** prepare a parallel release stack then switch traffic.
- **Canary:** send a small percentage to new version and watch metrics.

One instance on one host cannot guarantee zero downtime across a restart. Proper availability generally needs multiple ready instances.

## 6. Database changes and rollback
Deploy backward-compatible changes first: expand schema, migrate/backfill, then remove old fields once all replicas use new format. Rolling back an image cannot automatically reverse dropped data.

## 7. Operational tests
Simulate a 3-second HTTP request, send SIGTERM, observe that it finishes before shutdown. Repeat with a timeout. Monitor 5xx and p95 during a deployment and validate rollback behavior.

## 8. Common mistakes
Starting old-instance shutdown before new is ready; closing DB before requests finish; no deadline; forgetting queue workers and sockets; assuming payment retried safely without idempotency.

**Interview answer:** Graceful deployment uses readiness-aware routing, bounded draining, shutdown signals, backward-compatible data migrations and observability to replace instances without abruptly interrupting users.
