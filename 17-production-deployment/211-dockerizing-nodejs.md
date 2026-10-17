# Lesson 211 — Dockerizing a Node.js Application

## 1. What problem does Docker solve?
Your Express API runs locally but fails on another machine because its Node version, operating-system libraries or installed packages differ. Docker creates an **image** with the application and its runtime dependencies so it can run consistently.

**Image** = read-only blueprint. **Container** = running instance of an image. **Dockerfile** = instructions to build an image.

```text
Source + package-lock.json + Dockerfile
              |
         docker build
              v
         Docker image
              |
          docker run
              v
        Running container
```

## 2. Prepare a minimal Express app
`package.json`:
```json
{
  "type": "module",
  "scripts": { "start": "node src/server.js" },
  "dependencies": { "express": "^5.0.0" }
}
```
Install with `npm install` to generate and commit a matching `package-lock.json`.

`src/server.js`:
```js
import express from "express";
const app = express();
app.get("/health", (req, res) => res.json({ ok: true }));
const port = Number(process.env.PORT || 3000);
app.listen(port, "0.0.0.0", () => console.log("Listening", port));
```

## 3. First Dockerfile
```dockerfile
FROM node:22-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY src ./src
ENV NODE_ENV=production
USER node
EXPOSE 3000
CMD ["node", "src/server.js"]
```

**Line by line:**
- `FROM`: base image with Node.js.
- `WORKDIR`: app folder inside image.
- `COPY`: send only needed files.
- `RUN npm ci`: reproducible install from lockfile.
- `ENV`: production runtime mode.
- `USER node`: run without root privileges.
- `EXPOSE`: documents intended listening port; does not publish it.
- `CMD`: process started by the container.

Choose a currently supported Node version for actual deployment and pin an appropriate image digest where reproducibility is critical. The example uses Node 22 for consistency with your earlier projects.

## 4. .dockerignore
```text
node_modules
.git
.env
.env.*
!.env.example
coverage
logs
*.log
Dockerfile*
README.md
```
This reduces context size and avoids accidentally copying credentials. Keep files your build actually requires.

## 5. Build, run and test
```bash
docker build -t node-api:local .
docker run --rm -p 3000:3000 --name node-api node-api:local
curl http://localhost:3000/health
```

`-p 3000:3000` maps host port to container port. Bind explicitly to `127.0.0.1:3000:3000` if you want local-only host access.

## 6. Containers and databases
A container is an isolated process environment, **not a VM with a permanent filesystem**. Data stored only in its writable layer may disappear when the container is replaced. Use an external database, Docker volumes, or managed storage.

**Common mistake:** connecting to `localhost:27017` inside the API container and expecting the host's MongoDB. `localhost` points to the container itself. In Compose, connect to the DB using its **service name**, such as `mongodb:27017`.

## 7. Secrets and networking
Don't bake `.env`, database passwords, or API keys into images. Pass secrets at runtime via Compose environment/secrets or your deployment platform's secret manager. Environment variables are convenient, but not automatically encrypted.

## 8. Production checklist
Use a non-root user, small base image, lockfile, correct signal handling, health/readiness endpoints, graceful shutdown, image vulnerability scanning and a multi-stage build where compilation is required.

## 9. Common mistakes
Copying local `node_modules` into image; running as root unnecessarily; using `npm install` without lockfile for reproducibility; believing `EXPOSE` publishes ports; embedding credentials; not handling SIGTERM.

## Practice / interview
Build and run the sample API, then explain image vs container, `-p`, `WORKDIR`, `COPY`, and why `localhost` changes meaning inside Docker.

**Interview answer:** Docker packages an application with a consistent runtime. A Dockerfile builds the image, a container runs it, and production design includes reproducible dependencies, non-root execution, runtime secrets, networking, health checks and graceful shutdown.
