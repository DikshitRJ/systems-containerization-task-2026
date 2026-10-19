# Phase 1: End-to-End Build and Verification

## 1. Overview
The goal of Phase 1 was to get the entire multi-container application stack (PostgreSQL database, Node.js/Express backend, React/Vite frontend, and Nginx reverse proxy) building and running end-to-end using Docker Compose, diagnosing and resolving any errors encountered across application, container, and network layers.

---

## 2. Issues Encountered and Diagnostics

### Issue 1: Nginx Proxy Crash on Startup (Permission Denied on `default.conf`)
* **Layer**: Container / Host Mount / Security (SELinux)
* **Observed Symptom**:
  The `orbis-proxy` container repeatedly restarted immediately after launch.
* **Exact Log Output**:
  ```text
  2026/10/19 11:31:17 [emerg] 1#1: open() "/etc/nginx/conf.d/default.conf" failed (13: Permission denied) in /etc/nginx/nginx.conf:31
  nginx: [emerg] open() "/etc/nginx/conf.d/default.conf" failed (13: Permission denied) in /etc/nginx/nginx.conf:31
  ```
* **Root Cause Diagnosis**:
  Inspecting the host file using `ls -Z nginx/default.conf` revealed the SELinux context `unconfined_u:object_r:unlabeled_t:s0`. On Linux distributions with SELinux enforcing, container runtimes restrict container processes from reading host bind mounts unless the container-sharing label flag (`:z` or `:Z`) is appended to the volume declaration.
* **Resolution**:
  Updated the volume mount in [docker-compose.yml](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/docker-compose.yml):
  ```yaml
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro,z
  ```

---

### Issue 2: Nginx Upstream Routing Misconfiguration (`backend-api` vs `backend`)
* **Layer**: Network / Reverse Proxy Routing
* **Observed Symptom**:
  In [nginx/default.conf](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/nginx/default.conf), line 8 defined:
  `set $backend_service http://backend-api:4000;`
* **Root Cause Diagnosis**:
  In `docker-compose.yml`, the service name was defined as `backend` (and container name `orbis-backend`), not `backend-api`. Nginx's internal DNS resolver (`127.0.0.11`) could not resolve `backend-api`, which would result in HTTP 502 Bad Gateway (`Host not found`) for all requests routed to `/api/`.
* **Resolution**:
  Modified [nginx/default.conf](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/nginx/default.conf) to point to the actual service name:
  ```nginx
  location /api/ {
      set $backend_service http://backend:4000;
      proxy_pass $backend_service;
      ...
  ```

---

### Issue 3: Missing OpenSSL Runtime Library in Backend Alpine Image
* **Layer**: Application / Container Runtime
* **Observed Symptom**:
  The backend container continuously crashed and exited after 5 failed retries.
* **Exact Log Output**:
  ```text
  prisma:warn Prisma failed to detect the libssl/openssl version to use, and may not work as expected. Defaulting to "openssl-1.1.x".
  Please manually install OpenSSL and try installing Prisma again.
  Database connection attempt 1 failed: PrismaClientInitializationError: Unable to require('/app/node_modules/.prisma/client/libquery_engine-linux-musl.so.node').
  The Prisma engines do not seem to be compatible with your system. Please refer to the documentation about Prisma's system requirements: https://pris.ly/d/system-requirements

  Details: Error loading shared library libssl.so.1.1: No such file or directory (needed by /app/node_modules/.prisma/client/libquery_engine-linux-musl.so.node)
  Max retries reached. Exiting...
  ```
* **Root Cause Diagnosis**:
  The backend base image is `node:20-alpine`. Alpine Linux uses OpenSSL 3 by default and does not include `libssl.so.1.1` or the `openssl` CLI package out of the box. When `npx prisma generate` was executed during build, Prisma CLI could not detect the OpenSSL version and defaulted to the incompatible `openssl-1.1.x` binary engine target.
* **Resolution**:
  Added `RUN apk add --no-cache openssl` to [backend/Dockerfile](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/Dockerfile) before running `npx prisma generate`. This enables Prisma to automatically generate the compatible `linux-musl-openssl-3.0.x` query engine binary.

---

### Issue 4: Network Isolation Preventing Backend from Communicating with Database
* **Layer**: Container Networking
* **Observed Symptom**:
  The starter `docker-compose.yml` assigned `database` only to `backend-network`, while assigning `backend` only to `frontend-network`, and `proxy` to both networks.
* **Root Cause Diagnosis**:
  Docker bridge networks isolate containers. Because `backend` was not on `backend-network`, it could not resolve or connect to `database:5432`. Furthermore, `proxy` was inadvertently placed on `backend-network`, violating network isolation principles.
* **Resolution**:
  In [docker-compose.yml](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/docker-compose.yml):
  - Added `backend` to both `frontend-network` and `backend-network`.
  - Removed `proxy` from `backend-network`, keeping it strictly on `frontend-network`.
  - Added a healthcheck on `database` (`pg_isready -U postgres -d orbis`) and configured `backend` to depend on `database: { condition: service_healthy }`.

---

### Issue 5: Database Schema Migrations Not Applied on First Run
* **Layer**: Application / Database
* **Observed Symptom**:
  PostgreSQL started with a fresh, empty database `orbis`. Without applying the 20 Prisma migrations in `backend/prisma/migrations/`, any database queries (such as `GET /api/events`) failed because the tables did not exist.
* **Root Cause Diagnosis**:
  The starter backend Dockerfile ran `CMD ["npm", "start"]` (`node server.js`), which connects to the database but does not apply outstanding migrations.
* **Resolution**:
  Updated [backend/Dockerfile](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/Dockerfile) command to run migrations before starting the Node server:
  ```dockerfile
  CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
  ```

---

### Issue 6: Unhandled Reference Error during Backend Graceful Shutdown
* **Layer**: Application Code
* **Observed Symptom**:
  Inspecting [backend/server.js](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/server.js) revealed that line 40 called `app.listen(PORT, ...)` without storing the returned HTTP server instance in a variable, but line 53 called `server.close(...)`.
* **Root Cause Diagnosis**:
  Receiving SIGTERM/SIGINT during container shutdown would trigger `ReferenceError: server is not defined`, crashing the process uncleanly.
* **Resolution**:
  Updated line 40 in [backend/server.js](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/server.js) to:
  ```javascript
  const server = app.listen(PORT, () => {
  ```

---

### Issue 7: Frontend API URL Fallback Configuration
* **Layer**: Frontend Client / Build Environment
* **Observed Symptom**:
  Frontend components in `frontend/src/` make API requests to `${import.meta.env.VITE_API_URL}/api/...`. If `VITE_API_URL` is undefined during build, strings evaluate to `"undefined/api/..."`. If it defaults to `http://localhost:4000`, the browser attempts to bypass the reverse proxy.
* **Root Cause Diagnosis**:
  The architecture requires the browser to communicate exclusively through the Nginx reverse proxy on port 80/443.
* **Resolution**:
  - Set `ENV VITE_API_URL=""` in [frontend/Dockerfile](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/frontend/Dockerfile) so that all API requests use relative paths (`/api/...`).
  - Updated [frontend/src/api/api.js](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/frontend/src/api/api.js) to default `baseURL` to `''`.

---

## 3. End-to-End Verification Results

### Container Status (`docker compose ps`)
```text
NAME             IMAGE                                         COMMAND                  SERVICE    CREATED          STATUS                    PORTS
orbis-backend    systems-containerization-task-2026-backend    "docker-entrypoint.s…"   backend    25 seconds ago   Up 19 seconds             4000/tcp
orbis-db         postgres:15-alpine                            "docker-entrypoint.s…"   database   27 seconds ago   Up 24 seconds (healthy)   5432/tcp
orbis-frontend   systems-containerization-task-2026-frontend   "docker-entrypoint.s…"   frontend   27 seconds ago   Up 24 seconds             80/tcp
orbis-proxy      nginx:alpine                                  "/docker-entrypoint.…"   proxy      25 seconds ago   Up 17 seconds             0.0.0.0:80->80/tcp, [::]:80->80/tcp
```

### Frontend Proxy Verification (`curl -i http://localhost/`)
```text
HTTP/1.1 200 OK
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:35:54 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 452
Connection: keep-alive
...
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <title>Orbis</title>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

### Backend Proxy Verification (`curl -i http://localhost/api/events`)
```text
HTTP/1.1 200 OK
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:35:58 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 2
Connection: keep-alive
X-Powered-By: Express
...
[]
```

### Database Migration and Schema Verification (`psql`)
All 20 Prisma migrations applied automatically on startup, populating the database relations:
```text
               List of relations
 Schema |        Name        | Type  |  Owner   
--------+--------------------+-------+----------
 public | Application        | table | postgres
 public | ApplicationForm    | table | postgres
 public | CustomQuestion     | table | postgres
 public | Event              | table | postgres
 public | EventBranding      | table | postgres
 public | EventLink          | table | postgres
 public | EventPerson        | table | postgres
 public | EventTimeline      | table | postgres
 public | Prize              | table | postgres
 public | ProjectSubmission  | table | postgres
 public | Sponsor            | table | postgres
 public | Team               | table | postgres
 public | TeamMember         | table | postgres
 public | Track              | table | postgres
 public | User               | table | postgres
 public | UserEducation      | table | postgres
 public | UserExperience     | table | postgres
 public | UserProfile        | table | postgres
 public | UserSkill          | table | postgres
 public | UserSocialProfile  | table | postgres
 public | _prisma_migrations | table | postgres
(21 rows)
```
