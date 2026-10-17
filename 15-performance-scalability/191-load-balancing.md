# Lesson 191 — Load Balancing

## 1. What is a load balancer?
A **load balancer** accepts incoming traffic and forwards each request/connection to an appropriate backend instance.

```text
             Browser clients
                   |
                   v
             Load balancer
             /     |     \
         Node A  Node B  Node C
              \    |    /
              Shared DB/Redis
```

**Why useful:** distribute traffic, remove unhealthy instances, support rolling deployments and prevent one instance from becoming the only point of failure. The load balancer itself needs redundancy in serious production deployments.

## 2. Basic algorithms
| Algorithm | Simple meaning | Suitable when |
| --- | --- | --- |
| Round robin | Rotate requests among backends | Similar servers |
| Weighted round robin | Larger servers get larger share | Different capacity |
| Least connections | Choose backend with fewer active connections | Uneven request durations |
| Hash-based / consistent hashing | Map key to backend predictably | Some state/locality needs |
| Random / power of two choices | Pick backends with low overhead | Large distributed fleets |

No algorithm guarantees equal **CPU load** because requests differ in cost.

## 3. Practical Nginx example
Two Express servers:
- Node A on port `3001`
- Node B on port `3002`

**nginx.conf snippet**
```nginx
http {
  upstream node_api {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
  }

  server {
    listen 8080;
    location / {
      proxy_pass http://node_api;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

This is an **illustrative local config**, not a complete security-hardened production setup. It assumes both Node servers and Nginx are already running. Production should use HTTPS/TLS, appropriate timeouts, upstream health management, access controls and trusted proxy configuration.

## 4. How requests travel
1. Browser sends `GET /api/products` to Nginx.
2. Nginx chooses a backend using its configured balancing behavior.
3. Node handles request and queries MongoDB if needed.
4. Response returns through Nginx to browser.
5. If a backend fails, load-balancer health/failure handling should stop routing to it according to configuration.

## 5. Reverse proxy vs load balancing
A **reverse proxy** sits in front of server(s), handling routing, TLS, headers, caching or compression. **Load balancing** is one function a reverse proxy can perform. One reverse proxy can forward to a single server and not balance anything.

## 6. Proxy trust and real client IP
Behind a proxy, Express may see the proxy's IP instead of the user's IP. Set `app.set("trust proxy", ...)` to match your **actual trusted proxy topology**. Trusting arbitrary forwarded IP headers can allow spoofing and bypass IP-based rate limits.

## 7. Health checks, retries and timeouts
A production load balancer should avoid unhealthy backends. Configure request/idle/connect timeouts and graceful draining on deployment.

**Be careful with retries:** repeating a GET may be reasonable, but retrying a payment POST without an idempotency mechanism can cause duplicate charges. Some proxies automatically retry certain failures; inspect behavior.

## 8. WebSockets and Socket.IO
WebSocket upgrades require correct reverse-proxy headers/timeouts. Socket.IO polling may require sticky sessions across replicas, and broadcasts need a shared adapter (e.g., Redis) for cross-instance room events.

## 9. Load balancer vs application cache vs CDN
- Load balancer: decides which backend receives a request.
- Cache/Redis: avoids repeated computation/database work.
- CDN: distributes static/cachable content geographically.
These components solve different problems.

## 10. Common mistakes
Thinking a load balancer speeds a single slow database query; forgetting TLS or trusted proxy settings; relying on local sessions; not draining on deploys; unsafe automatic retries; ignoring long-lived connections.

## Practice
Run two Express apps returning their port, proxy through Nginx and observe request distribution. Stop one server and inspect errors/failover behavior. Draw an architecture combining load balancer, Redis, MongoDB and three API instances.

**Interview answer:** Load balancing distributes requests among instances to improve availability and capacity. Choose appropriate algorithms, health checks, trusted proxy settings, session strategy, retry rules and graceful shutdown.
