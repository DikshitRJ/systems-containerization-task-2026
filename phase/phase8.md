# Phase 8: Bonus — Container Hardening and CI Pipeline

## 1. Overview
The goal of Phase 8 is to enforce enterprise-grade container security best practices:
1. **Container Hardening**: Running container processes under non-privileged users (`USER`) rather than root, minimizing runtime attack surfaces, and pruning build dependencies.
2. **Build Context Protection**: Crafting comprehensive `.dockerignore` files across frontend, backend, and root workspaces to prevent secret leakage and context bloat.
3. **Automated CI Pipeline**: Establishing a GitHub Actions CI workflow (`.github/workflows/docker-build.yml`) that lints Dockerfiles with Hadolint and validates multi-container builds on all branch commits.

---

## 2. Container Hardening Implementation

### 1. Non-Root User Execution (`USER node`)
Running containers as root violates the principle of least privilege: if an application vulnerability (such as remote code execution) is exploited, the attacker obtains root capabilities within the container and potentially the host namespace.

In [backend/Dockerfile](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/Dockerfile):
* Leveraged the built-in Alpine `node` user (`uid=1000, gid=1000`).
* Used `COPY --chown=node:node` during the build stages to assign ownership directly to the unprivileged user without creating duplicate filesystem layers.
* Switched execution context via `USER node` before opening ports and running the process.

```dockerfile
FROM node:20-alpine

WORKDIR /app

RUN apk add --no-cache openssl

COPY --chown=node:node package*.json ./
COPY --chown=node:node prisma ./prisma/
RUN npm ci
RUN npx prisma generate

COPY --chown=node:node . .

USER node

EXPOSE 4000

CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
```

### Verification Output:
```bash
docker compose exec backend whoami
docker compose exec backend id
```

**Raw Output**:
```text
node
uid=1000(node) gid=1000(node) groups=1000(node),1000(node)
```
The backend process runs completely isolated from root privileges.

---

### 2. Eliminating Runtime Bloat and Build Tools
* **Frontend**: The multi-stage build completely eliminates Node.js, npm, package managers, and development source files from the production image. Only the compiled HTML/JS/CSS assets are placed inside an Alpine Nginx container.
* **Backend**: Used Alpine Linux with `--no-cache` package management to ensure transient package lists are not persisted to the final image.

---

## 3. Build Context Protection (`.dockerignore`)

Dedicated `.dockerignore` files prevent local files, dependency trees, and sensitive environment secrets from leaking into the Docker build context:

### [backend/.dockerignore](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/.dockerignore)
```text
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
Dockerfile
README.md
```

### [frontend/.dockerignore](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/frontend/.dockerignore)
```text
node_modules
dist
npm-debug.log
.git
.gitignore
.env
.env.*
Dockerfile
README.md
```

### Root [.dockerignore](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/.dockerignore)
```text
.git
.gitignore
node_modules
.env
phase/
*.md
```

---

## 4. GitHub Actions CI Workflow

A robust CI workflow was authored at [.github/workflows/docker-build.yml](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/.github/workflows/docker-build.yml):

```yaml
name: Docker Build & Lint CI

on:
  push:
    branches:
      - '**'
  pull_request:
    branches:
      - '**'

jobs:
  lint:
    name: Lint Dockerfiles
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Lint Backend Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: backend/Dockerfile
          failure-threshold: error

      - name: Lint Frontend Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: frontend/Dockerfile
          failure-threshold: error

  build:
    name: Build Multi-Container Images
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate self-signed SSL certificates for testing
        run: |
          mkdir -p nginx/certs
          openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
            -keyout nginx/certs/server.key \
            -out nginx/certs/server.crt \
            -subj "/CN=localhost/O=Orbis/C=US" \
            -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

      - name: Setup environment file
        run: cp .env.example .env

      - name: Build Docker Compose Stack
        run: docker compose build
```

### Pipeline Features:
1. **Trigger Scope**: Triggers across all branches on both `push` and `pull_request`.
2. **Phase 1: Linting**: Employs Hadolint to validate Dockerfile syntax, maintainability rules, and security guidelines.
3. **Phase 2: Compilation & Image Build**: Configures Docker Buildx, creates test certificates, copies `.env.example`, and builds the full stack to verify zero build errors.
