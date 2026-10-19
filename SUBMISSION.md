# Systems Containerization Task 2026 - Final Submission

**Candidate**: Dikshit RJ  
**Domain**: Containers, Docker, Docker Compose, Nginx, PostgreSQL, Express, React  
**Repository**: [WebClub-NITK/systems-containerization-task-2026](https://github.com/WebClub-NITK/systems-containerization-task-2026)

---

## Table of Contents
1. [Reproduction Instructions](#1-reproduction-instructions)
2. [Phase 1: End-to-End Build and Verification](#2-phase-1-end-to-end-build-and-verification)
3. [Phase 2: Frontend Image Optimization](#3-phase-2-frontend-image-optimization)
4. [Phase 3: Database Configuration and Volume Persistence](#4-phase-3-database-configuration-and-volume-persistence)
5. [Phase 4: Network Segmentation and Isolation](#5-phase-4-network-segmentation-and-isolation)
6. [Phase 5: Nginx SSL Termination and HTTPS Redirection](#6-phase-5-nginx-ssl-termination-and-https-redirection)
7. [Phase 6: Consolidated Orchestration and Build Cache Analysis](#7-phase-6-consolidated-orchestration-and-build-cache-analysis)
8. [Phase 7: Bonus — Scaling, Load Balancing, and Rate Limiting](#8-phase-7-bonus--scaling-load-balancing-and-rate-limiting)
9. [Phase 8: Bonus — Container Hardening and CI Pipeline](#9-phase-8-bonus--container-hardening-and-ci-pipeline)
10. [Summary of Changes](#10-summary-of-changes)

---

## 1. Reproduction Instructions

To start the entire multi-container application stack from a clean state:

### Step 1: Clone Repository and Navigate to Workspace
```bash
git clone https://github.com/WebClub-NITK/systems-containerization-task-2026.git
cd systems-containerization-task-2026
```

### Step 2: Configure Environment Variables
Copy the template environment file:
```bash
cp .env.example .env
```

### Step 3: Generate SSL Certificates
Ensure the SSL certificate and private key are generated for local HTTPS:
```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/CN=localhost/O=Orbis/C=US" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

### Step 4: Build and Start Containers
Launch all services via Docker Compose in single command:
```bash
docker compose up -d --build
```

### Step 5: Verify Running Services
```bash
docker compose ps
```

### Step 6: Access the Application
* **Frontend**: Open `http://localhost` (automatically redirects to `https://localhost`)
* **API Endpoints**: Open `https://localhost/api/events`

---

## 2. Phase 1: End-to-End Build and Verification
*Detailed documentation*: [phase/phase1.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase1.md)

### Key Issues Identified & Resolved:
1. **Nginx SELinux Mount Permission Denied**:
   - *Log*: `open() "/etc/nginx/conf.d/default.conf" failed (13: Permission denied)`
   - *Fix*: Appended `:ro,z` to volume bind mount in `docker-compose.yml`.
2. **Nginx Upstream Host Mismatch**:
   - *Log*: `set $backend_service http://backend-api:4000;` failed with `Host not found`.
   - *Fix*: Updated `nginx/default.conf` upstream target to `backend:4000`.
3. **Missing OpenSSL in Backend Alpine Image**:
   - *Log*: `PrismaClientInitializationError: Error loading shared library libssl.so.1.1`
   - *Fix*: Added `RUN apk add --no-cache openssl` in `backend/Dockerfile`.
4. **Network Partition Between Backend and Database**:
   - *Cause*: `backend` was placed solely on `frontend-network`, while `database` was solely on `backend-network`.
   - *Fix*: Connected `backend` to both networks and removed `proxy` from `backend-network`.
5. **Database Schema Initialization**:
   - *Fix*: Configured startup entrypoint to run `npx prisma migrate deploy` with PostgreSQL healthcheck.
6. **Graceful Shutdown Crash in Backend**:
   - *Cause*: `app.listen` return value was not assigned to `server`.
   - *Fix*: Updated to `const server = app.listen(...)`.

---

## 3. Phase 2: Frontend Image Optimization
*Detailed documentation*: [phase/phase2.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase2.md)

* **Initial Size**: 1.29 GB (due to full Debian `node:20` base, full devDependencies in runtime, and `npx serve`).
* **Optimization**: Converted to multi-stage build:
  1. Build stage: `node:20-alpine` compiling React/Vite SPA.
  2. Runtime stage: `nginx:alpine` serving compiled static assets from `/usr/share/nginx/html`.
* **Final Size**: **70 MB** (Well below the 110MB ceiling; **-94.6% reduction**).

---

## 4. Phase 3: Database Configuration and Volume Persistence
*Detailed documentation*: [phase/phase3.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase3.md)

* **Environment-Driven Configuration**: Configured exclusively via `${POSTGRES_USER}`, `${POSTGRES_PASSWORD}`, `${POSTGRES_DB}`, and `${DATABASE_URL}` without hardcoded strings.
* **Port Isolation**: Database port 5432 is not published to the host (`nc -zv 127.0.0.1 5432` returns `Connection refused`).
* **Volume Persistence**: Inserted test record `test-user-phase3`, destroyed containers via `docker compose down`, recreated via `docker compose up -d`, and verified data remained 100% intact.

---

## 5. Phase 4: Network Segmentation and Isolation
*Detailed documentation*: [phase/phase4.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase4.md)

* **Network Topology**:
  - `frontend-network`: Connects `orbis-proxy`, `orbis-frontend`, and `orbis-backend`.
  - `backend-network`: Connects strictly `orbis-backend` and `orbis-db`.
* **Demonstrated Isolation**:
  - Host cannot reach backend (4000) or database (5432).
  - Frontend container cannot resolve or reach database (`nc: bad address 'database'`).
  - Reverse proxy container cannot resolve or reach database (`nc: bad address 'database'`).
  - Backend communicates directly with database (`database (172.21.0.2:5432) open`).

---

## 6. Phase 5: Nginx SSL Termination and HTTPS Redirection
*Detailed documentation*: [phase/phase5.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase5.md)

* **SSL Termination**: Nginx terminates TLS 1.3 / TLS 1.2 on port 443 with modern cipher suites.
* **HTTP Redirection**: All incoming port 80 requests return `HTTP/1.1 301 Moved Permanently` to `https://$host$request_uri`.
* **Proxy Decryption**: Decrypted requests are proxied internally to `frontend:80` and `backend:4000` with `X-Forwarded-Proto https`.

---

## 7. Phase 6: Consolidated Orchestration and Build Cache Analysis
*Detailed documentation*: [phase/phase6.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase6.md)

* **Orchestration**: Single-command startup via `docker compose up -d --build`.
* **Secrets Management**: Dynamic extraction into `.env` and `.env.example`.
* **Layer Caching Analysis**:
  - Instruction ordering placed `COPY package*.json` and `RUN npm ci` before `COPY . .`.
  - Modifying a single line in `server.js` re-used cache for steps 1 through 7 (`CACHED`).
  - Only the final `COPY . .` layer rebuilt, reducing rebuild duration to **2.8 seconds**.

---

## 8. Phase 7: Bonus — Scaling, Load Balancing, and Rate Limiting
*Detailed documentation*: [phase/phase7.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase7.md)

* **Horizontal Scaling**: Scaled backend service across 3 container replicas (`systems-containerization-task-2026-backend-[1,2,3]`).
* **Nginx Upstream Pool**: Load balances requests across backend replicas with persistent keepalive connections.
* **Rate Limiting**: Enforced client IP-based rate limiting (`limit_req_zone $binary_remote_addr zone=api_limit:10m rate=200r/s`). Verified burst protection returning `HTTP 429 Too Many Requests`.
* **Benchmarking (ApacheBench)**:
  - Single instance: 302.17 requests/sec (mean 33.09 ms).
  - Scaled setup: 235.37 requests/sec (mean 42.49 ms).
  - High availability and failover verified under load without connection drops.

---

## 9. Phase 8: Bonus — Container Hardening and CI Pipeline
*Detailed documentation*: [phase/phase8.md](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/phase/phase8.md)

* **Non-Root User Configuration**:
  - Backend runs under unprivileged `node` user (`USER node`, `uid=1000, gid=1000`).
  - Files copied with `--chown=node:node` to eliminate duplicate permission layers.
* **Context Leak Prevention**: Comprehensive `.dockerignore` files prevent local files, `.git`, `node_modules`, and `.env` secrets from entering Docker build contexts.
* **GitHub Actions Workflow**: `.github/workflows/docker-build.yml` triggers on all branch commits to lint Dockerfiles via Hadolint and validate multi-container builds.

---

## 10. Summary of Changes

| Component | File | Summary of Changes |
| :--- | :--- | :--- |
| **Compose** | `docker-compose.yml` | Fixed network segmentation, added healthcheck & service conditions, parameterized credentials from `.env`, added port 443, SSL cert mounts, enabled horizontal replica scaling. |
| **Backend** | `backend/Dockerfile` | Added OpenSSL runtime dependency for Prisma, optimized instruction ordering for layer caching, configured `USER node` non-root execution with `--chown=node:node`, added automated startup migration deployment. |
| **Backend** | `backend/server.js` | Fixed graceful shutdown by assigning `app.listen()` to `const server`. |
| **Frontend** | `frontend/Dockerfile` | Implemented multi-stage build (`node:20-alpine` builder + `nginx:alpine` runtime) reducing image size from 1.29GB to 70MB (< 110MB limit). |
| **Frontend** | `frontend/nginx.conf` | Created custom SPA routing configuration with client-side history mode support and immutable asset caching. |
| **Frontend** | `frontend/src/api/api.js` | Configured default API baseURL to relative path `''` to ensure all browser requests route through reverse proxy. |
| **Proxy** | `nginx/default.conf` | Implemented HTTP-to-HTTPS redirect (port 80 to 443), SSL termination (TLS 1.2/1.3), upstream load balancing pool (`backend_cluster`), rate limiting zone with burst control, and security headers. |
| **Security** | `nginx/certs/` | Generated 2048-bit RSA key and X.509 SSL certificate for localhost SAN. |
| **Configuration** | `.env`, `.env.example` | Centralized environment variables and secrets for clean single-command orchestration. |
| **Context** | `.dockerignore` files | Created root, backend, and frontend `.dockerignore` files preventing leaks of secrets and build bloat. |
| **CI/CD** | `.github/workflows/docker-build.yml` | Created automated GitHub Actions pipeline linting Dockerfiles with Hadolint and validating multi-container builds. |
| **Documentation** | `phase/phase[1-8].md` | Comprehensive, human-language documentation for every individual phase. |
