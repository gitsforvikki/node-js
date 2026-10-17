# Lesson 205 — Redis Pub/Sub

## 1. What is Pub/Sub?
**Publish/Subscribe** lets a publisher send a message to a channel; all **currently connected subscribers** to that channel receive it.

```text
Publisher -> Redis channel "notifications"
                    |
                    +--> Subscriber A (connected) -> receives
                    +--> Subscriber B (connected) -> receives
                    +--> Subscriber C (offline)   -> misses
```

Redis Pub/Sub is **fire-and-forget**: no durable storage, consumer acknowledgment or offline replay. It is different from Redis Streams and BullMQ.

## 2. Real use cases
- Notify active API instances that cache should be invalidated.
- Forward ephemeral presence or typing events.
- Broadcast short live updates between real-time servers.

Don't rely on Pub/Sub alone for critical payment processing, durable order work or messages a disconnected user must receive later.

## 3. Run Redis and install client
```bash
docker run --rm --name pubsub-demo -p 6379:6379 redis:7
npm install redis
```
Use `"type": "module"` for ESM.

## 4. Runnable subscriber
**subscriber.js**
```js
import { createClient } from "redis";

const subscriber = createClient({ url: "redis://127.0.0.1:6379" });
subscriber.on("error", console.error);
await subscriber.connect();

await subscriber.subscribe("notifications", (message, channel) => {
  console.log("Received from", channel, JSON.parse(message));
});
console.log("Waiting for notifications...");
```

## 5. Runnable publisher
**publisher.js**
```js
import { createClient } from "redis";

const publisher = createClient({ url: "redis://127.0.0.1:6379" });
publisher.on("error", console.error);
await publisher.connect();

const count = await publisher.publish(
  "notifications",
  JSON.stringify({ type: "typing", conversationId: "conv-1" })
);
console.log("Subscribers reached:", count);
await publisher.quit();
```

Start subscriber first, then publisher. `count` reports how many subscribed client connections Redis published to, **not confirmed processing**.

## 6. Why use separate connections?
A Redis connection dedicated to subscribing has different command behavior; maintain a separate publisher client or duplicate the connected client:
```js
const sub = publisher.duplicate();
await sub.connect();
```

## 7. Pub/Sub vs Streams vs BullMQ

| Technology | Durable pending work? | Best for |
| --- | --- | --- |
| Redis Pub/Sub | No | Ephemeral live fan-out |
| Redis Streams | Can retain entries + consumer acknowledgment | Event/message processing |
| BullMQ | Queue job states, retries and scheduling using Redis | Background tasks |

Durability still depends on Redis persistence, retention, deployment and consumer logic; none magically gives exactly-once business effects.

## 8. Channel design and security
Use clear names like `chat:room:123` and `cache:products:invalidate`. Avoid embedding secrets in payloads. Validate JSON and message schemas, authenticate Redis access and use private network/TLS where appropriate.

Redis channel names **do not authorize end users**. If a Pub/Sub message becomes a Socket.IO event, the API must still enforce room/recipient permissions.

## 9. Failure and reconnect
If Redis disconnects briefly, a subscriber misses messages published during the outage. For durable chat notifications, save authoritative state in MongoDB and let reconnecting clients fetch missed events. For critical work use Streams/queues with acknowledgment.

## 10. Common mistakes and practice
Mistakes: using Pub/Sub as an email job queue; assuming every publish reaches offline consumers; treating subscriber count as delivery confirmation; sharing private user events with broad channels; forgetting Redis errors and reconnection behavior.

Practice: run two subscribers. Publish once and confirm both receive it. Stop one subscriber, publish again, reconnect it and observe that the missed event isn't replayed.

**Interview answer:** Redis Pub/Sub broadcasts messages to active subscribers with low latency, but provides no persistence or acknowledgments. Use it for ephemeral live events and cache invalidation; use Streams or a queue for recoverable work.
