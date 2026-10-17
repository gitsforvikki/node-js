# Lesson 199 — Scheduled Jobs

## 1. What is a scheduled job?
A scheduled job runs **at a chosen time or interval**, rather than immediately after an HTTP request.

Examples:
- Every morning: email an application follow-up digest.
- Every night: remove expired temporary uploads.
- Every hour: generate an analytics report.
- In 30 minutes: send one reminder.

## 2. Scheduled vs delayed vs repeated

| Type | Meaning |
| --- | --- |
| Immediate | Run as soon as a worker is free |
| Delayed | Run once, no earlier than a specified delay |
| Repeated | Run according to a recurring interval/calendar rule |
| Cron | Calendar-based schedule, like every day at 09:00 |

**Important:** A scheduler triggers work; a worker executes it. They may run in separate processes.

## 3. Simple cron expression

```text
minute hour day-of-month month day-of-week
   0     9        *          *       *
```

This commonly means **09:00 every day** in the scheduler's configured time zone. Day-of-week and daylight-saving behavior vary by scheduler and time zone; configure explicitly.

## 4. Node-cron example
Install `npm install node-cron`.

```js
import cron from "node-cron";

cron.schedule("0 9 * * *", async () => {
  try {
    console.log("Run daily summary");
    await generateDailySummary();
  } catch (error) {
    console.error("Scheduled task failed", error);
  }
}, { timezone: "Asia/Kolkata", noOverlap: true });
```

This assumes `generateDailySummary()` exists and uses a node-cron version supporting these options. **Process-local schedules stop when the process stops**; they don't automatically catch up on missed runs. Running this in three API replicas can run the task three times.

## 5. BullMQ delayed job
```js
await reminderQueue.add("follow-up", {
  applicationId: "app-123",
}, {
  delay: 30 * 60 * 1000,
  jobId: "follow-up-app-123",
});
```

Delay measures minimum wait, **not exact guaranteed execution time**. A worker must be available and Redis functional. Bound retained jobs and choose retry behavior.

## 6. BullMQ Job Scheduler (modern BullMQ)

```js
await reportQueue.upsertJobScheduler(
  "daily-report",
  { pattern: "0 9 * * *", tz: "Asia/Kolkata" },
  {
    name: "generate-report",
    data: { reportType: "daily" },
    opts: { attempts: 3 },
  }
);
```

This uses a BullMQ Job Scheduler API; check the installed release's documentation for supported scheduling options and time zones. Prefer `upsertJobScheduler()` over older deprecated repeatable-job APIs in modern BullMQ.

## 7. Distributed deployment: avoid duplicates
In Kubernetes/PM2/container replicas, don't register the same **in-process cron** everywhere unless multiple executions are intentional. Use one dedicated scheduler, a distributed leader election/lock, a managed scheduler, or BullMQ scheduling with suitable guarantees. Even then, processing may repeat—make jobs idempotent.

## 8. What about missed and long-running jobs?
A scheduling system needs policies for:
- downtime and missed triggers;
- overlapping runs;
- changing time zones/DST;
- worker outages;
- retries and partial completion;
- cancellation/rescheduling.

“Run daily at 9” does not guarantee execution at exactly 9:00:00.

## 9. Practice
Schedule a cleanup every night in your local time zone. Explain what happens if the app is down at trigger time and how to avoid three replicas running the same task.

**Interview answer:** Scheduled jobs enqueue or execute work at specified times. Cron is calendar-based, delayed jobs are one-time, and durable schedulers coordinate multiple instances. Always plan time zones, overlaps, missed runs, retries and idempotency.
