# Lesson 215 — Nginx Basics for Node.js

## 1. Why Nginx?
Nginx is a web server and reverse proxy that can accept public HTTP(S) traffic and forward it to one or more Express applications.

```text
Browser --> Nginx :80/:443 --> Express :3000
                               |
                              MongoDB
```

## 2. Basic configuration concepts
- `server`: a virtual host listening on an IP/port.
- `location`: choose how to handle a URL path.
- `proxy_pass`: forward the request to an upstream service.
- `upstream`: group backend servers for load balancing.

## 3. Local Node.js proxy configuration
First, start Express on port 3000.

```nginx
events {}

http {
  server {
    listen 8080;
    server_name localhost;

    location / {
      proxy_pass http://127.0.0.1:3000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

Save as a local `nginx.conf`, then run Nginx with that configuration using your installed binary/container. Confirm that `http://localhost:8080` reaches Express.

## 4. Test/reload configuration
```bash
nginx -t
# On a system-managed Nginx install:
sudo systemctl reload nginx
```
Use the path to your active config if necessary: `nginx -t -c /path/to/nginx.conf`. Reloading avoids a full stop when the configuration is valid.

## 5. Load-balance two Express servers
```nginx
upstream node_backends {
  server 127.0.0.1:3001;
  server 127.0.0.1:3002;
}

server {
  listen 8080;
  location / {
    proxy_pass http://node_backends;
    proxy_set_header Host $host;
  }
}
```
This snippet belongs **inside an `http` block**. Start two apps on the specified ports; Nginx will distribute incoming requests according to its upstream algorithm (round robin by default).

## 6. WebSocket / Socket.IO proxy
```nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {
  listen 8080;
  location /socket.io/ {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_read_timeout 75s;
  }
}
```
Put `map` in the `http` context, not directly inside `server`. Tune timeouts for heartbeat settings. Multi-instance Socket.IO may additionally need sticky routing and a Redis adapter.

## 7. HTTPS in production
Use a domain and valid TLS certificate from your deployment's certificate provider/CA; redirect HTTP to HTTPS, configure certificates/renewal and HSTS only when ready. Don't ship the HTTP-only examples above as a public secure deployment.

## 8. Proxy body limits, caching, access logs
For uploads, Nginx's `client_max_body_size` must align with app limits. Log status/latency while avoiding private query strings and Authorization headers. Cache only responses safe for shared caches; never publicly cache private user data by mistake.

## 9. Common mistakes
Editing the wrong active config; forgetting Nginx reload; wrong `proxy_pass` slash; missing WebSocket headers; thinking Nginx authorizes user records; configuring `trust proxy` unsafely.

## Practice
Run Nginx locally on 8080 with Express on 3000. Add second Express instance, configure upstream, and show which process handled each request.

**Interview answer:** Nginx can route and load-balance requests in front of Node.js, terminate TLS and support WebSockets. Configure server/location/upstream directives carefully and align proxy security, timeouts and forwarding headers.
