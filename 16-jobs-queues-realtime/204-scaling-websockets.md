# Lesson 204 — Scaling WebSocket Applications

## 1. Why scaling changes real-time systems
One Socket.IO instance knows its own connections and rooms. When load balancing distributes clients across two Node.js servers, a message handled on Server A must still reach a recipient connected to Server B.

```text
Client A --> Load balancer --> Node A
                                   |
                               Redis adapter
                                   |
Client B --> Load balancer --> Node B
```

Without a distributed adapter, `io.to(room).emit(...)` may reach only sockets on the emitting process.

## 2. Two separate concerns
**Connection routing:** Does the client reconnect/upgrade to a compatible backend?  
**Event fan-out:** Can all backend instances broadcast to sockets connected elsewhere?

A Redis adapter helps with **event fan-out**. It doesn't automatically solve sticky sessions for transports requiring repeated HTTP polling requests.

## 3. Socket.IO Redis adapter example
Install:
```bash
npm install socket.io @socket.io/redis-adapter redis
```
Assume an existing `io` instance and a secure Redis deployment.

```js
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

pubClient.on("error", console.error);
subClient.on("error", console.error);

await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// Then normal room emits work across adapter-connected Socket.IO servers.
io.to("conversation:conv-1").emit("chat:new", message);
```

Set up the adapter **before accepting production connections** and close Redis clients during graceful shutdown.

## 4. Sticky sessions and transports
Socket.IO often supports HTTP long-polling and WebSocket. Polling uses multiple requests that may need to reach the same process, requiring **sticky sessions** at the load balancer. If you choose WebSocket-only transport on both sides, the transport requirement may change, but you lose polling fallback and must configure proxy upgrades/timeouts correctly.

## 5. Redis Pub/Sub does not persist chat
The classic Redis adapter uses Pub/Sub for forwarding broadcasts. Messages published during an adapter outage may not reach remote sockets. It **does not replace MongoDB chat history** or application replay logic.

Reliable chat pattern:
1. Verify sender membership.
2. Persist message in MongoDB.
3. Emit event to currently connected room members.
4. On reconnect, client fetches missed messages using a cursor/last message ID.
5. Deduplicate message creation using a client-generated idempotency key where appropriate.

## 6. Presence across servers
A process-local `onlineUsers` Map is incomplete. Use shared presence records with expiring heartbeats, per-device identifiers and cleanup; a user with two active sockets should not become "offline" when just one disconnects.

## 7. Scaling limits
- Number of open sockets and file descriptors.
- Memory per connection.
- Outbound message bandwidth and slow consumers.
- Load balancer idle timeouts / ping intervals.
- Redis throughput and pub/sub fan-out.
- MongoDB writes and indexes.
- Reconnect storms after deployments/network outages.

Measure connection count, event latency, backpressure, reconnect rate, error rate and Redis health.

## 8. Production security
Authenticate handshake and authorize **every event**. Restrict origin/handshake, bound payload size and rate, validate user IDs, use `wss://`, and avoid sending secrets through broad rooms.

## 9. Common mistakes
Adding replicas without Redis adapter; thinking Redis Pub/Sub is durable; assuming sticky sessions solve room broadcast; relying on `socket.id` for user identity; keeping message history only in RAM.

## Practice
Run two Socket.IO processes on different ports behind a development reverse proxy. Connect one client to each and confirm room broadcasts cross instances with the Redis adapter.

**Interview answer:** Scaling WebSockets requires load-balancer compatibility, shared event fan-out, externalized state, durable message history and reconnect replay. A Redis adapter distributes live events, but it isn't an offline inbox.
