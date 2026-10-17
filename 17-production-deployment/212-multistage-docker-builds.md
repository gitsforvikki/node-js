# Lesson 212 — Multi-stage Docker Builds

## 1. Why use multiple stages?
A TypeScript backend needs compiler tooling during **build**, but not in the final runtime container. Shipping `typescript`, tests, caches and build utilities increases image size and attack surface.

Multi-stage builds use separate stages to build and run the app:

```text
Stage 1: dependencies + compiler -> build dist/
Stage 2: production dependencies + dist/ -> smaller runtime image
```

## 2. Runnable TypeScript app
Typical scripts:
```json
{
  "type": "module",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js"
  }
}
```
Install runtime dependencies and `typescript` as a dev dependency. Make sure your `tsconfig.json` compiles `src/server.ts` into `dist/server.js`; ESM imports should be compatible with emitted Node.js code.

## 3. Multi-stage Dockerfile
```dockerfile
# Stage 1 — build
FROM node:22-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# Stage 2 — production
FROM node:22-bookworm-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

## 4. Explain each important line
`AS build` names a temporary stage. `RUN npm ci` installs dev tools for compilation. `COPY --from=build` copies the compiled app, **not the TypeScript source/compiler**. The final image starts with only runtime dependencies and output.

This approach installs dependencies twice in build context but avoids shipping build-time packages. Native modules must be compatible with the final runtime platform.

## 5. Why COPY package files first?
Docker can reuse a cached dependency-install layer when only source code changes. If you copy all source files first, small edits may invalidate dependency installation and make builds slower.

## 6. Test
```bash
docker build -t ts-api:local .
docker run --rm -p 3000:3000 ts-api:local
```
Check the application's health endpoint. The actual app must listen on `0.0.0.0` inside the container.

## 7. Security and supply chain
- Pin base images to a reviewed digest for strict reproducibility.
- Update Node runtime and OS packages on a schedule.
- Avoid passing secrets via build arguments; build logs/layers can leak them.
- Use `.dockerignore` to exclude `.env`, `node_modules`, test fixtures and secrets.
- Scan the **final** runtime image, not only the build stage.
- Ensure final image contains everything required at runtime, such as schema assets or templates.

## 8. Common mistakes
Omitting `dist` from runtime; using dev-only runtime imports; mixing incompatible native builds; copying secrets; assuming multi-stage automatically ensures zero vulnerabilities; leaving root as runtime user.

## Practice
Compare `docker images` sizes for single-stage and multi-stage TypeScript app. Verify `node_modules/typescript` is absent from final image.

**Interview answer:** A multi-stage Docker build compiles software using build dependencies in one stage and copies only runtime artifacts into a smaller final image. It improves reproducibility, reduces image size and limits unnecessary tooling.
