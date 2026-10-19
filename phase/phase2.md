# Phase 2: Frontend Image Optimization

## 1. Overview
The goal of Phase 2 is to inspect the frontend Dockerfile, identify sources of container image bloat, and optimize the build process so that the final frontend image remains strictly under **110MB**.

---

## 2. Initial Assessment

### Initial Image Analysis
The starter frontend Dockerfile was defined as follows:
```dockerfile
FROM node:20

WORKDIR /app

COPY . .
RUN npm install
RUN npm run build

EXPOSE 80
CMD ["npx", "serve", "-s", "dist", "-l", "80"]
```

Checking the resulting image size using `docker images`:
```text
REPOSITORY                                    TAG         IMAGE ID       SIZE
systems-containerization-task-2026-frontend   latest      1b1950c7afbb   1.29GB
```

### Why Was the Starter Image So Large (1.29GB)?
1. **Bulky Base Image**: `node:20` is based on Debian Linux, containing a full GNU C library, build toolchains, and package managers (~1.1GB uncompressed base).
2. **Lingering Node Modules and Build Artifacts**: The starter Dockerfile copied all project files, ran `npm install` (downloading full development dependencies), and kept `node_modules/` (~180MB) inside the final runtime container.
3. **Runtime Overhead**: Serving static production files via `npx serve` requires the entire Node.js runtime and npm package execution environment, none of which is required once Vite has compiled the application into static HTML, JavaScript, and CSS bundles in the `dist` directory.

---

## 3. Optimization Architecture: Multi-Stage Build

To achieve an image size strictly below the 110MB threshold, we transitioned to a **multi-stage Docker build** using a specialized web server (`nginx:alpine`) for the production runtime:

### Stage 1: Builder Stage (`node:20-alpine`)
* Uses a lightweight Alpine Linux Node image (~170MB during build).
* Isolates `package.json` and `package-lock.json` before code copying to leverage Docker layer caching.
* Runs `npm install` and compiles the React/Tailwind application via Vite into static assets in `/app/dist`.
* This entire stage is discarded after compilation; none of its build tools or `node_modules` leak into the final image.

### Stage 2: Production Runtime Stage (`nginx:alpine`)
* Uses the battle-tested, ultra-lightweight `nginx:alpine` image (~40MB base).
* Copies only the compiled static output from `/app/dist` into `/usr/share/nginx/html`.
* Includes a custom, optimized [frontend/nginx.conf](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/frontend/nginx.conf) enabling:
  - HTML5 History Mode / SPA routing fallback (`try_files $uri $uri/ /index.html;`) so direct URL navigation works without 404 errors.
  - Aggressive client-side caching for immutable hashed assets in `/assets/` (`max-age=31536000, immutable`).

---

## 4. Optimized Configuration

### [frontend/Dockerfile](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/frontend/Dockerfile)
```dockerfile
# Stage 1: Build stage
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
ENV VITE_API_URL=""
RUN npm run build

# Stage 2: Minimal production runtime stage
FROM nginx:alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### [frontend/nginx.conf](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/frontend/nginx.conf)
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    location /assets/ {
        root /usr/share/nginx/html;
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

---

## 5. Verification and Size Comparison

### Final Image Size (`docker images`)
```text
REPOSITORY                                    TAG             SIZE
systems-containerization-task-2026-frontend   latest          70MB
```

### Size Comparison Table
| Metric | Initial Image | Constraint Target | Optimized Image | Reduction |
| :--- | :--- | :--- | :--- | :--- |
| **Image Size** | **1.29 GB** | **< 110 MB** | **70 MB** | **-94.6%** |

### Runtime Verification (`curl -i http://localhost/`)
```text
HTTP/1.1 200 OK
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:39:16 GMT
Content-Type: text/html
Content-Length: 452
Connection: keep-alive
Last-Modified: Mon, 19 Oct 2026 11:38:50 GMT
ETag: "6ad6014a-1c4"
Accept-Ranges: bytes

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Orbis</title>
    <script type="module" crossorigin src="/assets/index-BcDuHabI.js"></script>
    <link rel="stylesheet" crossorigin href="/assets/index-C8H1LJcJ.css">
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```
The frontend container successfully builds at **70MB**, well below the 110MB ceiling, and serves the Single Page Application through the reverse proxy.
