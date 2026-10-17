# Lesson 214 — Reverse Proxy Concepts

## 1. What is a reverse proxy?
A **reverse proxy** is a server in front of your Node.js API. Clients connect to the proxy rather than directly to each Node process.

```text
Internet client
     |
     v
Nginx / CDN / load balancer (reverse proxy)
     |
     +--> Express A
     +--> Express B
```

A **forward proxy** acts on behalf of clients; a reverse proxy acts on behalf of servers.

## 2. Why use it?
- Terminate HTTPS/TLS at the edge.
- Route `/api` to Express, `/static` to a static server.
- Load-balance multiple backend instances.
- Apply body/time limits, access policies and caching.
- Handle WebSocket upgrades and keep-alive connections.

A reverse proxy doesn't automatically fix insecure controller logic or database queries.

## 3. Request flow
1. Browser requests `https://example.com/api/products`.
2. Nginx accepts TLS connection.
3. Nginx forwards the HTTP request to internal Express `http://api:3000`.
4. Express responds; proxy forwards the response to client.

The backend may see the proxy's IP unless forwarded headers are correctly configured.

## 4. Minimal route example
```nginx
server {
  listen 80;
  server_name localhost;

  location /api/ {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```
This is a local HTTP example; production must serve HTTPS and apply security controls. Because `proxy_pass` has **no trailing URI slash**, the `/api/` path is preserved; slash choices affect URL rewriting.

## 5. Express trust proxy
If Express runs behind a trusted proxy:
```js
app.set("trust proxy", "loopback"); // ONLY when proxy is trusted loopback
```
This affects `req.ip`, protocol and other proxy-derived information. Configure for your **actual deployment's proxy addresses/hops**; blindly using `true` may trust spoofed `X-Forwarded-For` from clients.

## 6. Proxy vs load balancer vs API gateway
- Reverse proxy: forwards requests and can do TLS/routing.
- Load balancer: distributes requests among several backends.
- API gateway: often adds auth, quotas, service discovery and API-specific policy.
One product can perform more than one role.

## 7. Proxy timeouts and buffering
Long API requests may time out at Nginx even when Node is still working. Configure connection/read timeouts intentionally. Large uploads may be buffered or rejected by proxy body-size rules; align limits across Nginx and Express.

## 8. WebSockets
Real-time chat needs an HTTP Upgrade-compatible proxy and appropriate idle timeouts/heartbeats. Socket.IO polling may also need sticky sessions when scaled horizontally.

## 9. Common mistakes
Directly exposing app port publicly; trusting forwarded headers from arbitrary clients; accidental path rewrite; logging private tokens in URLs; missing WebSocket Upgrade headers; retrying unsafe payment POSTs without idempotency.

## Practice
Explain how Express receives `req.ip` behind Nginx, and trace a GET request through browser → proxy → API → MongoDB → response.

**Interview answer:** A reverse proxy sits before backend services to handle routing, TLS, buffering and sometimes load balancing. Configure trusted forwarding headers and request limits; keep authentication and authorization in the backend.
