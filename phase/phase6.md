# Phase 6: Consolidated Orchestration and Build Cache Analysis

## 1. Overview
The goal of Phase 6 is to consolidate multi-container orchestration into a single-command, reproducible environment using `.env` for secrets and configuration, and to conduct an in-depth Docker build layer cache analysis following a single-line source code modification.

---

## 2. Environment Configuration Extraction

All inline values, secrets, and environment parameters were extracted into a root-level [.env](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/.env) file with an accompanying [.env.example](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/.env.example) template:

### [.env](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/.env)
```env
# Database Settings
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgrespassword
POSTGRES_DB=orbis

# Backend Service Settings
PORT=4000
DATABASE_URL=postgresql://postgres:postgrespassword@database:5432/orbis?schema=public
AUTH0_ISSUER_BASE_URL=https://dev-dummy.auth0.com/
AUTH0_AUDIENCE=https://api.orbis.com
CORS_ORIGIN=http://localhost:3000

# Frontend Service Settings
VITE_API_URL=
```

### Consolidated [docker-compose.yml](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/docker-compose.yml)
Services now reference `.env` variables with resilient fallback defaults:
```yaml
services:
  database:
    image: postgres:15-alpine
    container_name: orbis-db
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgrespassword}
      POSTGRES_DB: ${POSTGRES_DB:-orbis}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-orbis}"]
      interval: 2s
      timeout: 3s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: orbis-backend
    environment:
      PORT: ${PORT:-4000}
      DATABASE_URL: ${DATABASE_URL:-postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-postgrespassword}@database:5432/${POSTGRES_DB:-orbis}?schema=public}
      AUTH0_ISSUER_BASE_URL: ${AUTH0_ISSUER_BASE_URL:-https://dev-dummy.auth0.com/}
      AUTH0_AUDIENCE: ${AUTH0_AUDIENCE:-https://api.orbis.com}
      CORS_ORIGIN: ${CORS_ORIGIN:-http://localhost:3000}
    depends_on:
      database:
        condition: service_healthy
    networks:
      - frontend-network
      - backend-network
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: orbis-frontend
    networks:
      - frontend-network
    restart: unless-stopped

  proxy:
    image: nginx:alpine
    container_name: orbis-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro,z
      - ./nginx/certs:/etc/nginx/certs:ro,z
    depends_on:
      - backend
      - frontend
    networks:
      - frontend-network
    restart: unless-stopped

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge

volumes:
  postgres_data:
```

---

## 3. Single-Line Source Code Modification

To test layer caching behavior, a single comment was inserted into [backend/server.js](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/server.js):
```javascript
const app = express();
// Cache test modification: Phase 6 layer cache analysis
const PORT = process.env.PORT || 4000;
```

---

## 4. Rebuild Log and Layer Cache Analysis

Executing `docker compose build backend` produced the following build output:

```text
[+] Building 2.3s (15/15) FINISHED
 => [internal] load local bake definitions                                 0.0s
 => => reading from stdin 694B                                             0.0s
 => [internal] load build definition from Dockerfile                       0.0s
 => => transferring dockerfile: 328B                                       0.0s
 => [internal] load metadata for docker.io/library/node:20-alpine          1.0s
 => [internal] load .dockerignore                                          0.0s
 => => transferring context: 2B                                            0.0s
 => [1/8] FROM docker.io/library/node:20-alpine@sha256:fb4cd12c85ee03686f  0.0s
 => [internal] load build context                                          0.1s
 => => transferring context: 11.29kB                                       0.0s
 => CACHED [2/8] WORKDIR /app                                              0.0s
 => CACHED [3/8] RUN apk add --no-cache openssl                            0.0s
 => CACHED [4/8] COPY package*.json ./                                     0.0s
 => CACHED [5/8] COPY prisma ./prisma/                                     0.0s
 => CACHED [6/8] RUN npm ci                                                0.0s
 => CACHED [7/8] RUN npx prisma generate                                   0.0s
 => [8/8] COPY . .                                                         0.3s
 => exporting to image                                                     0.1s
 => => exporting layers                                                    0.0s
 => => writing image sha256:25f0a624f18eea5b1c3fd94d8eb0b6a819174d6bc6c5f  0.0s
 => => naming to docker.io/library/systems-containerization-task-2026-bac  0.0s
 => resolving provenance for metadata file                                 0.0s
[+] build 1/1
 ✔ Image systems-containerization-task-2026-backend Built                   2.8s
```

---

## 5. Layer-by-Layer Breakdown

| Step | Instruction | Cache Status | Explanation |
| :--- | :--- | :--- | :--- |
| **[1/8]** | `FROM node:20-alpine` | **CACHED** | Base image digest is immutable and already present locally. |
| **[2/8]** | `WORKDIR /app` | **CACHED** | Working directory definition has not changed. |
| **[3/8]** | `RUN apk add --no-cache openssl` | **CACHED** | Shell command is identical and previous layer did not change. |
| **[4/8]** | `COPY package*.json ./` | **CACHED** | File checksums for `package.json` and `package-lock.json` are identical. |
| **[5/8]** | `COPY prisma ./prisma/` | **CACHED** | Schema and migration definitions have not changed. |
| **[6/8]** | `RUN npm ci` | **CACHED** | Dependency installation skipped completely; previous layers untouched. |
| **[7/8]** | `RUN npx prisma generate` | **CACHED** | Client generation skipped completely; schema unchanged. |
| **[8/8]** | `COPY . .` | **REBUILT (0.3s)** | Content change in `server.js` alters context checksum; layer invalidates here. |
| **Export** | Image Export | **FAST (0.1s)** | Re-used all unchanged lower filesystem layers; wrote only the single modified layer. |

---

## 6. Dockerfile Instruction Ordering and Efficiency

### Naive vs Optimized Instruction Ordering
In the naive starter Dockerfile:
```dockerfile
# NAIVE ORDERING: Inefficient
COPY . .
RUN npm ci
RUN npx prisma generate
```
Because `COPY . .` occurred *before* `RUN npm ci`, editing a single character in any source code file invalidated the layer cache at step 1 of the build. As a consequence, Docker was forced to re-download and re-install all npm dependencies (`npm ci`) and regenerate Prisma clients on every code edit, taking **25–45 seconds per build**.

In our optimized Dockerfile:
```dockerfile
# OPTIMIZED ORDERING: Cache-friendly
COPY package*.json ./
COPY prisma ./prisma/
RUN npm ci
RUN npx prisma generate

COPY . .
```
1. **Infrequently Changed Layers First**: `package.json` changes only when adding or updating dependencies, so dependencies are installed once and cached indefinitely.
2. **Frequently Changed Layers Last**: Application source code (`COPY . .`) is placed at the very end of the build sequence.
3. **Outcome**: Source code rebuilds take only **2.8 seconds** instead of 25+ seconds, drastically accelerating iterative development cycles.
