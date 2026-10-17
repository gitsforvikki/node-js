# Lesson 213 — Docker Compose

## 1. Why Compose?
An Express project may require **API + MongoDB + Redis**. Starting each container manually with many `docker run` commands is error-prone. Docker Compose declares services, networks, environment and volumes in one YAML file.

```text
Browser -> API service (port 3000)
             |        |
             v        v
          MongoDB    Redis
         (volume)   (cache)
```

## 2. Working compose.yaml example
Assume `Dockerfile` from Lesson 211 and an API that uses `MONGODB_URI` and `REDIS_URL`.

```yaml
services:
  api:
    build: .
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      NODE_ENV: production
      PORT: 3000
      MONGODB_URI: mongodb://mongodb:27017/shophub
      REDIS_URL: redis://redis:6379
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  mongodb:
    image: mongo:7
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand({ping:1}).ok"]
      interval: 10s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  mongo_data:
```

This is a **local development example** with no MongoDB or Redis authentication and only the API published to the host loopback interface. Do **not** expose these unprotected services publicly. Production deployments require protected credentials, backup/retention, private networking and appropriately managed databases.

## 3. Run commands
```bash
docker compose up --build -d
docker compose ps
docker compose logs -f api
docker compose down
```
`docker compose down` removes the containers and Compose network. `docker compose down -v` also deletes named volumes, including **MongoDB data**—do not use `-v` casually.

## 4. Why service names work
Inside Docker's Compose network, `mongodb` and `redis` resolve to the corresponding services. `localhost` inside API points to **API itself**, not MongoDB.

```text
API container connects:
mongodb://mongodb:27017/shophub  ✓
mongodb://localhost:27017/...    ✗ (unless MongoDB lives in API container)
```

## 5. depends_on limitations
`depends_on` with `service_healthy` waits for the declared health check before starting the dependent service. It does **not** guarantee the DB will stay healthy forever. Your API must still handle later disconnects/retries.

## 6. Volumes vs bind mounts
**Named volume:** Docker-managed persistent data (`mongo_data`). **Bind mount:** maps host path into container (useful for local source hot-reload). Avoid exposing secrets or writable host filesystem unnecessarily.

## 7. Override for development
You can use `compose.override.yaml` for bind mounts and `npm run dev` without changing a production-oriented Dockerfile. Do not bind-mount host `node_modules` over container dependencies accidentally.

## 8. Environment variables and secrets
Compose variable substitution (from environment or `.env`) differs from injecting variables into containers. Do not commit credentials. For production, use managed secrets or approved secret mechanisms, restrict permissions and rotate keys.

## 9. Common mistakes
Confusing `localhost`; deleting volumes with `down -v`; publishing Redis/MongoDB to the internet; assuming `depends_on` handles runtime failures; mixing development bind mounts with production deployments.

## Practice
Run API/MongoDB/Redis; verify name resolution, stop Redis and observe API error handling, then restart Redis without losing MongoDB data.

**Interview answer:** Docker Compose describes a multi-container application and shared network, letting services resolve each other by name. Use health checks, persistent volumes and secure runtime configuration; it is especially useful for local development and integration testing.
