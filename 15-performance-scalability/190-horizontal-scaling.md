# Lesson 190 — Horizontal Scaling

## 1. What is scaling?
**Scaling** increases the system's ability to handle more traffic or work.

- **Vertical scaling:** give one server more CPU/RAM.
- **Horizontal scaling:** run more independent app instances and distribute traffic among them.

```text
             Load balancer
               /      \
        Express A     Express B
            \          /
                MongoDB
                   |
                 Redis
```

## 2. Real-world example
Your CodeBuddy-like chat app starts with one Express server. As traffic rises, you deploy three containers. All three must:
- read/write the same MongoDB data;
- validate the same centrally managed authentication sessions/tokens;
- use a shared rate-limit or session store if needed;
- coordinate WebSocket broadcasts/rooms across instances.

Otherwise User A on server 1 might not see an event generated on server 2.

## 3. Why stateless HTTP handlers are easier
**Stateless** doesn't mean your application has no data. It means a given API instance does **not** rely on exclusive in-process session state between requests.

Bad:
```js
const sessions = new Map(); // only this one server knows these sessions
```
Better: shared authenticated session store, or properly verified access tokens with a planned revocation strategy, with durable user data in a shared DB.

## 4. Scaling bottlenecks don't disappear
Increasing API containers can overload shared dependencies:
- More app instances means more **database pool connections**.
- Redis can become a hotspot.
- External APIs impose quotas.
- Queue workers may process duplicates unless jobs are idempotent.
- Cost rises with instance count.

Example: 5 containers × `maxPoolSize: 20` can permit up to ~100 app connections for a relevant DB pool (plus topology/monitoring overhead). Database limits still apply.

## 5. Horizontal scaling vs clustering
Cluster runs multiple Node worker processes on one machine. Horizontal scaling adds separately deployable app instances (often on multiple machines). Both increase independent execution capacity; horizontal scaling also improves fault isolation across hosts.

## 6. Autoscaling
An autoscaler may add replicas when CPU, requests, latency or queue depth crosses a target. It should also scale down carefully to avoid flapping.

CPU is not enough for every workload: a slow database can cause high request latency while API CPU remains low. Monitor saturation and user-visible errors too.

## 7. Production checklist
- Shared durable database and migration strategy.
- Externalized sessions/cache/queues as required.
- Health endpoints and graceful shutdown.
- Proper deployment readiness/rolling updates.
- Centralized logs, metrics and request IDs.
- Rate limits/quotas enforced across instances.
- Idempotency for critical writes and background jobs.
- Resource limits and cost monitoring.

## 8. Consistency and sticky sessions
A sticky session routes a user repeatedly to one instance, but can create uneven load and complicate failover. Prefer properly shared state when feasible. Sticky routing may still be required for some WebSocket transports/architectures.

## 9. Common mistakes
Adding replicas before checking DB indexes; local `Map` used as shared cache; file uploads stored only on container disk; global rate limits enforced per instance; no graceful shutdown; assuming 3 replicas always triple throughput.

## Practice
Draw a three-replica Express + MongoDB + Redis architecture. Decide where sessions, uploads and chat room events live. Explain what happens if one instance dies.

**Interview answer:** Horizontal scaling adds app instances and uses load balancing to distribute traffic. It requires shared state, attention to bottleneck dependencies, reliable deployment and observability; more instances do not automatically increase throughput linearly.
