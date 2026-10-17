# Lesson 157 — Graceful Shutdown

## 1. What is graceful shutdown?

A server may need to stop because of a deployment, container restart, Ctrl+C, or fatal error. **Graceful shutdown** means it stops accepting new requests, lets existing work finish within a deadline, closes resources, and exits.

Without it, a restart during checkout may interrupt a request or leave work incomplete.

```text
SIGTERM / SIGINT
   -> readiness becomes false
   -> stop accepting new connections
   -> allow in-flight work (bounded)
   -> close DB / queue / other resources
   -> exit
```

## 2. SIGINT vs SIGTERM

- `SIGINT`: commonly from Ctrl+C in terminal.
- `SIGTERM`: commonly sent by process managers, containers, and deployment systems.

These are **signals**, not Express routes. Handle them once at the process level.

## 3. Working Express + Mongoose example

```js
import express from "express";
import mongoose from "mongoose";

const app = express();
let shuttingDown = false;

app.get("/health/live", (req, res) => res.sendStatus(200));
app.get("/health/ready", (req, res) => {
  res.sendStatus(!shuttingDown && mongoose.connection.readyState === 1 ? 200 : 503);
});

async function start() {
  await mongoose.connect(process.env.MONGODB_URI);
  const server = app.listen(3000, () => console.log("Server started"));

  async function shutdown(signal) {
    if (shuttingDown) return;
    shuttingDown = true;
    console.log(`Received ${signal}. Shutting down...`);

    // Hard deadline if something refuses to finish.
    const timer = setTimeout(() => {
      console.error("Shutdown deadline exceeded");
      process.exit(1);
    }, 15_000);
    timer.unref();

    try {
      await new Promise((resolve, reject) => {
        server.close((err) => err ? reject(err) : resolve());
      });
      await mongoose.disconnect();
      clearTimeout(timer);
      console.log("Shutdown complete");
      process.exitCode = 0;
    } catch (error) {
      clearTimeout(timer);
      console.error("Shutdown failed:", error);
      process.exitCode = 1;
    }
  }

  process.once("SIGTERM", () => { void shutdown("SIGTERM"); });
  process.once("SIGINT", () => { void shutdown("SIGINT"); });
}

start().catch((error) => {
  console.error("Startup failed:", error);
  process.exitCode = 1;
});
```

**Assumptions:** A valid `MONGODB_URI` is provided, no additional resources (Redis, WebSocket, workers) need closing, and your deployment's shutdown grace period is longer than the 15-second deadline.

## 4. Why is server.close important?

`server.close()` stops accepting new connections and waits for active HTTP requests to complete. Modern supported Node.js versions also close idle connections as part of this behavior.

If an active request never finishes, shutdown can hang. The deadline ensures eventual termination. Force-closing all connections may interrupt users; use it only after a deliberate grace period or in emergencies.

## 5. Readiness vs liveness

- **Liveness:** Is the process running and able to respond?
- **Readiness:** Should the load balancer send it **new traffic**?

During shutdown, readiness should return 503 while the process may remain live briefly to finish work.

In real orchestrated deployments, allow enough time for readiness changes to propagate **before** stopping the listener when necessary. The simple example above closes immediately; deployment-specific draining may require a short delay.

## 6. What about ongoing background jobs?

If you use BullMQ, a queue consumer, Socket.IO, or scheduled tasks, shutdown must also:
- stop accepting new jobs,
- finish/abort work within a deadline,
- close queue and Redis connections,
- close WebSockets cleanly,
- make retries safe using idempotency.

Do not assume closing Express automatically closes everything else.

## 7. Why not immediately process.exit()?

`process.exit()` can terminate before pending logs or I/O flush. Prefer setting `process.exitCode` after cleanup and allowing the event loop to drain, with a hard deadline for hangs.

For a fatal programmer error, exit **non-zero**. For a normal controlled shutdown, exit zero when cleanup succeeds.

## 8. Database transaction example

Suppose a payment was authorized but your order update is still in progress. A shutdown might interrupt it. Graceful shutdown reduces that risk but **does not guarantee exactly-once completion**.

Use transactions where supported, payment-provider idempotency keys, durable queues, and recovery procedures. A shutdown timer alone is not enough.

## 9. Test it locally

1. Start the app.
2. Send a request that intentionally waits 3 seconds.
3. In another terminal run `kill -TERM <PID>`.
4. Observe that new connections stop and existing work can finish.
5. Verify DB disconnect and process exit.
6. Repeat with a request longer than the shutdown deadline to see why a timeout matters.

## 10. Common mistakes

- Closing MongoDB before in-flight requests finish.
- No shutdown deadline, causing hung deployments.
- Ignoring Redis, WebSockets, queues, and scheduled jobs.
- Reporting ready while shutting down.
- Treating a crash as a clean success.
- Assuming graceful shutdown guarantees no lost work.

## 11. Interview answer

Graceful shutdown handles termination signals by stopping new work, draining active requests within a timeout, closing dependencies, and exiting with the right status. It reduces downtime and interrupted operations but must be paired with idempotency and durable job processing for critical workflows.
