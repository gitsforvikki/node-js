# Lesson 144 — Rate Limiting in Node.js APIs

## 1. What is rate limiting?

**Rate limiting** controls how many requests a client can make in a given time. Imagine a login API receiving 5,000 attempts in one minute. Without limits, it can overload the server and help attackers guess passwords.

A rule might be: **at most 100 requests per IP in 15 minutes**. Once the limit is exceeded, the API responds with **429 Too Many Requests**.

```text
Client -> API -> rate-limit check
                  |-- below limit -> route handler -> response
                  |-- over limit  -> HTTP 429
```

It protects availability and reduces abuse; it **does not replace authentication, authorization, or proper infrastructure DDoS protection**.

## 2. Rate limiting vs throttling vs concurrency limiting

- **Rate limiting:** how many requests over time (e.g., 100 per 15 minutes).
- **Throttling:** deliberately slow or pace incoming work.
- **Concurrency limiting:** how many operations run at the same moment (e.g., 20 database queries).
- **Request size limiting:** how large a body may be. This solves a different problem.

Example: 100 requests in 15 minutes can still arrive in one second. A rate limit alone may not prevent a burst from exhausting resources; concurrency limits and queues can help.

## 3. Implement it in Express 5

Install:

```bash
npm install express express-rate-limit
```

```js
import express from "express";
import { rateLimit } from "express-rate-limit";

const app = express();

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  limit: 100,                // maximum 100 requests per window
  standardHeaders: "draft-8",
  legacyHeaders: false,
  message: { message: "Too many requests. Try again later." },
});

app.use("/api", apiLimiter);
app.use(express.json({ limit: "16kb" }));

app.get("/api/products", (req, res) => {
  res.json({ products: [] });
});

app.listen(3000);
```

**Try it:** Temporarily set `limit: 3`, call `GET /api/products` four times from the same client, and observe that the fourth request gets 429.

```bash
for i in 1 2 3 4; do curl -i http://localhost:3000/api/products; done
```

The package's default store keeps counters in **one Node.js process**. It is fine for local demos, but not reliable across multiple replicas/restarts.

## 4. Use different limits for different endpoints

Not every route has equal risk:

| Endpoint | Example policy | Why |
| --- | --- | --- |
| GET /api/products | 100/min/IP | Reduce general API abuse |
| POST /auth/login | 5–10/min/IP plus account-based checks | Slow password guessing |
| POST /auth/forgot-password | Low limit per account + IP | Avoid email flooding |
| POST /api/checkout | Low, business-aware limits | Avoid duplicate work |

Numbers are **starting examples, not universal production defaults**. Tune from traffic, user experience, and risk.

```js
const loginLimiter = rateLimit({
  windowMs: 60_000,
  limit: 5,
  standardHeaders: "draft-8",
  legacyHeaders: false,
});

app.post("/auth/login", loginLimiter, loginController);
```

Place limiting **before** expensive password hashing, DB queries, and external API calls. A login limiter alone does not stop attacks spread across many IPs.

## 5. How does a limit identify a client?

A common key is the **IP address**. But users may share an IP (office, college, mobile network), and attackers may rotate IPs.

For authenticated routes, you can also use a trusted user ID. For login, use both IP-based and **account-based** defenses (Lesson 145). Never trust an arbitrary user ID header supplied by the client.

**Proxy warning:** In production, Express may sit behind Nginx, Render, or a cloud load balancer. Configure `trust proxy` for your **actual trusted proxy hops/network**. Never blindly trust all forwarded IP headers, because attackers may spoof them.

## 6. Multi-server deployments and Redis

```text
Client requests
      |
      v
Load balancer
   |       |
 Node A  Node B
   \       /
   Shared Redis counter store
```

If Node A and Node B have separate in-memory counters, a client may exceed the intended overall limit by splitting requests. A **shared store** such as Redis lets instances use common counters.

Use a Redis-compatible store supported by your installed rate-limiter version, with connection/error handling and TTL. Choose whether to **fail open** (allow traffic when Redis is down) or **fail closed** (reject traffic) based on endpoint risk. Test the behavior.

## 7. Algorithms in simple words

- **Fixed window:** count requests in fixed periods. Simple, but bursts may happen at boundaries.
- **Sliding window:** count requests over a moving time range. Smoother behavior.
- **Token bucket:** requests spend tokens that refill over time; allows controlled bursts.
- **Leaky bucket:** requests are processed at a steadier pace.

The right algorithm depends on the desired burst behavior and infrastructure. The library/store may use its own approach.

## 8. HTTP response and client handling

When over limit, return **429** with a short, non-sensitive message. Where supported, include standardized rate-limit information or `Retry-After`.

Frontend example:

```js
const res = await fetch("/api/products");
if (res.status === 429) {
  console.log("Please wait before trying again.");
}
```

Clients should use bounded retries/backoff, not an immediate retry loop. Avoid exposing sensitive account existence through different error messages.

## 9. Common mistakes

1. Using only one global limit for every endpoint.
2. Assuming in-memory limits work across multiple containers.
3. Trusting unverified `X-Forwarded-For` headers.
4. Limiting only by IP and ignoring distributed login attempts.
5. Treating rate limiting as a substitute for bot detection, WAF, or DDoS controls.
6. Setting limits so low that shared-IP users cannot use the app.

## 10. Practice and interview questions

1. Add separate limits for `GET /api/products` and `POST /auth/login`.
2. Explain why a Redis store is useful when running three Node.js replicas.
3. What does HTTP 429 mean?
4. Why is IP-based limiting alone insufficient against credential stuffing?

**Interview answer:** Rate limiting restricts request frequency to protect APIs from abuse and overload. In Express, use middleware like `express-rate-limit`, configure endpoint-specific policies, use a shared store for multiple instances, and combine it with authentication and abuse monitoring.
