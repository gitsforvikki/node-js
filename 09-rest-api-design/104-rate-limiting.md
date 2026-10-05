# Lesson 104 — Rate Limiting

## Why rate limiting matters

Rate limiting protects APIs from:

- abuse
- brute-force attacks
- accidental client loops
- scraping
- resource exhaustion
- expensive endpoint misuse

It is both a reliability and security control.

---

## 1. What is rate limiting?

Rate limiting restricts how many requests a client can make within a period.

Example:

```text
100 requests / minute
```

After that:

```text
429 Too Many Requests
```

---

## 2. What identifies a client?

Possible keys:

- IP address
- user ID
- API key
- tenant ID
- route + user
- organization ID

The right key depends on the API.

---

## 3. IP-based limiting

Simple:

```text
key = client IP
```

Useful for:
- public endpoints
- anonymous login attempts

Problems:
- shared NAT
- mobile networks
- proxies
- attacker IP rotation

---

## 4. User-based limiting

Authenticated API:

```text
key = userId
```

More accurate for logged-in clients.

---

## 5. API-key limiting

Public developer APIs often rate-limit per API key.

This enables:
- usage plans
- quotas
- billing tiers

---

# Rate-limiting Algorithms

## 6. Fixed window

Example:

```text
100 requests
per 60-second window
```

Simple.

Problem:

Client can send:

```text
100 requests at 12:00:59
+
100 requests at 12:01:00
```

Effectively 200 requests in 2 seconds.

---

## 7. Sliding window log

Store timestamps of requests and count requests in the last N seconds.

Accurate but memory-expensive at high scale.

---

## 8. Sliding window counter

Approximate sliding behavior using counters from adjacent windows.

Better efficiency than storing every timestamp.

---

## 9. Token bucket

Bucket contains tokens.

Each request consumes one.

Tokens refill over time.

```text
Bucket: 10 tokens

Request -> 9
Request -> 8

time passes
  |
  v
tokens refill
```

Allows controlled bursts.

---

## 10. Leaky bucket

Requests enter a queue/bucket and leave at a controlled rate.

Useful for smoothing traffic.

Conceptually:

```text
bursty input
   |
   v
bucket
   |
   v
steady output
```

---

## 11. Which algorithm is best?

Depends on needs.

Fixed window:
- simple

Token bucket:
- common
- burst-friendly

Sliding window:
- more precise fairness

Leaky bucket:
- smooth output

---

## 12. 429 response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

Body:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests"
  }
}
```

---

## 13. Useful rate-limit headers

APIs may expose standardized or conventional headers showing:

- limit
- remaining
- reset time

Follow current standards/library behavior.

Do not invent inconsistent header names without reason.

---

## 14. Login rate limiting

Login should be stricter.

Example:

```text
5 failed attempts / 15 minutes
```

Potential keys:
- IP
- account/email
- combination of both

Using only one dimension can be bypassed or harm legitimate users.

---

## 15. Password reset

Very important to limit:

```text
POST /forgot-password
```

Otherwise attackers can:
- spam email
- enumerate accounts
- create provider cost

---

## 16. Expensive endpoint limiting

Example:

```text
POST /generate-report
```

may be far more expensive than:

```text
GET /health
```

Use route-specific limits.

---

## 17. Global vs endpoint limits

Global:

```text
1000 requests/min
```

Route-specific:

```text
/login -> 5/min
/search -> 60/min
/export -> 2/min
```

This is often more effective.

---

## 18. In-memory rate limiting

Single process:

```text
memory counter
```

works for development or a single instance.

But with multiple instances:

```text
Instance A counter = 50
Instance B counter = 40
```

Client effectively gets 90.

Distributed apps need shared state.

---

## 19. Redis-based limiting

Shared Redis:

```text
Instance A
Instance B
Instance C
    |
    v
 Redis
```

Now all instances see the same counters.

---

## 20. Atomicity

Rate-limit counters must be updated atomically.

Bad:

```text
read count
increment in app
write count
```

Concurrent requests can race.

Use atomic Redis operations/scripts/transactions.

---

## 21. Reverse proxy/CDN rate limiting

Rate limiting can also happen at:

- CDN
- API gateway
- Nginx
- cloud load balancer

This blocks abusive traffic before it reaches Node.js.

---

## 22. Defense in depth

Good architecture may use:

```text
CDN / Gateway limit
        |
        v
App-level route limit
        |
        v
Business quota
```

Different layers solve different problems.

---

## 23. Rate limiting vs quota

Rate limit:

```text
100 requests/minute
```

Quota:

```text
10,000 requests/month
```

Different concepts.

Both may exist together.

---

## 24. Rate limiting vs concurrency limiting

Rate limit:

```text
requests per time
```

Concurrency limit:

```text
max requests processing simultaneously
```

An export endpoint might need both.

---

## 25. Distributed system caveat

Strict global rate limiting across regions can be expensive because of coordination.

Sometimes systems accept approximate regional limits for performance.

Trade-off:
- global precision
- latency/availability

---

## 26. Trust proxy

If using IP-based limits behind a reverse proxy, configure proxy trust correctly.

Otherwise:

```text
all clients appear to come from proxy IP
```

or malicious forwarded headers may be trusted incorrectly.

---

## 27. Do not block event loop

Rate-limit storage checks should be efficient.

Do not do:
- slow synchronous disk operations
- heavy computation
- full DB scans

in middleware.

---

## 28. Bypass strategy

Some internal/health endpoints may need different rules.

But avoid broad bypass rules such as:

```text
if internal header exists -> unlimited
```

unless the header is strongly authenticated.

---

## 29. Common mistakes

### Mistake 1
Only global one-size-fits-all limit.

### Mistake 2
In-memory limit in multi-instance production.

### Mistake 3
No atomic updates.

### Mistake 4
Wrong IP behind proxy.

### Mistake 5
No stricter auth endpoint protection.

### Mistake 6
Confusing rate limit with quota.

---

## 30. Interview questions

### What is rate limiting?

Controlling how many requests a client can make within a defined period.

### Which status code?

429 Too Many Requests.

### Why use Redis in distributed apps?

To maintain shared counters across multiple instances.

### Token bucket vs fixed window?

Fixed window is simpler; token bucket supports controlled bursts more naturally.

### Why route-specific limits?

Different endpoints have different cost and security risk.

---

## 31. Strong interview answer

> Rate limiting protects APIs from abuse and overload by limiting request frequency per IP, user, API key, or tenant. For simple single-instance systems, in-memory counters can work, but distributed services usually need a shared atomic store such as Redis or gateway-level enforcement. I apply stricter limits to sensitive or expensive endpoints like login, password reset, search, and report generation, return 429 with retry guidance, and choose an algorithm such as token bucket when controlled bursts are acceptable.

---

## Interview-Ready Summary

```text
Rate Limit
   |
   +--> IP
   +--> user
   +--> API key
   +--> tenant

Algorithms:
fixed window
sliding window
token bucket
leaky bucket

Distributed:
Redis / gateway

Response:
429 + Retry-After
```

## Practice Task

Design rate limits for:

- login
- forgot password
- product search
- checkout
- report generation
- public API key

Choose:
- key
- algorithm
- limit
- storage
