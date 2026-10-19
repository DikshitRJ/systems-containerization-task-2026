# Phase 3: Database Configuration and Volume Persistence

## 1. Overview
The goal of Phase 3 is to ensure that the PostgreSQL database container is configured exclusively via environment variables without hardcoded credentials, that its port remains strictly internal to the container network (not accessible directly from the host), and that data persists across container destruction and recreation using a named Docker volume (`postgres_data`).

---

## 2. Environment Variable Configuration

All database credentials and connection parameters are extracted from hardcoded strings and declared dynamically using environment variables in [docker-compose.yml](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/docker-compose.yml):

```yaml
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
    ...
    environment:
      PORT: ${PORT:-4000}
      DATABASE_URL: postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-postgrespassword}@database:5432/${POSTGRES_DB:-orbis}?schema=public
```

Neither [backend/Dockerfile](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/Dockerfile) nor the backend source files ([backend/src/config/database.js](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/backend/src/config/database.js)) contain hardcoded database credentials; connection strings are consumed strictly through `process.env.DATABASE_URL`.

---

## 3. Host Port Isolation Verification

The database service defines no `ports:` directive mapping container port `5432` to the host machine. It resides exclusively on `backend-network`.

### Verification Test from Host
Testing direct access to port 5432 from the host:
```bash
nc -zv -w 2 127.0.0.1 5432
```

### Raw Terminal Output
```text
nc: connect to 127.0.0.1 port 5432 (tcp) failed: Connection refused
```
Port 5432 is completely inaccessible from outside the internal Docker container network, preventing unauthorized external access to the database.

---

## 4. Volume Persistence Verification

Data persistence is managed through the named volume `postgres_data`, mounted at `/var/lib/postgresql/data` within the PostgreSQL container.

### Step 1: Insert Test Record into Database
Inserted a test record into the `User` table:
```bash
docker compose exec database psql -U postgres -d orbis -c "INSERT INTO \"User\" (\"id\", \"auth0Id\", \"email\", \"username\", \"role\", \"status\", \"createdAt\", \"updatedAt\") VALUES ('test-user-phase3', 'auth0|test-phase3', 'persistence@example.com', 'persistence_tester', 'ORGANIZER', 'ACTIVE', NOW(), NOW());"
```

**Output**:
```text
INSERT 0 1
```

### Step 2: Query Prior to Container Destruction
```bash
docker compose exec database psql -U postgres -d orbis -c "SELECT \"id\", \"email\", \"username\", \"role\" FROM \"User\" WHERE \"id\" = 'test-user-phase3';"
```

**Output**:
```text
        id        |          email          |      username      |   role    
------------------+-------------------------+--------------------+-----------
 test-user-phase3 | persistence@example.com | persistence_tester | ORGANIZER
(1 row)
```

### Step 3: Destroy Containers and Networks (`docker compose down`)
```bash
docker compose down
```

**Output**:
```text
[+] down 6/6
 ✔ Container orbis-proxy                                       Removed      0.3s
 ✔ Container orbis-backend                                     Removed      0.4s
 ✔ Container orbis-frontend                                    Removed      0.3s
 ✔ Container orbis-db                                          Removed      0.2s
 ✔ Network systems-containerization-task-2026_frontend-network Removed      0.1s
 ✔ Network systems-containerization-task-2026_backend-network  Removed      0.1s
```
All four containers and both bridge networks were completely destroyed.

### Step 4: Recreate Stack (`docker compose up -d`)
```bash
docker compose up -d
```

**Output**:
```text
[+] up 6/6
 ✔ Network systems-containerization-task-2026_frontend-network Created      0.1s
 ✔ Network systems-containerization-task-2026_backend-network  Created      0.2s
 ✔ Container orbis-frontend                                    Started      1.7s
 ✔ Container orbis-db                                          Healthy      4.5s
 ✔ Container orbis-backend                                     Started      5.6s
 ✔ Container orbis-proxy                                       Started      7.0s
```

### Step 5: Verify Data Persistence Across Destruction
```bash
docker compose exec database psql -U postgres -d orbis -c "SELECT \"id\", \"email\", \"username\", \"role\" FROM \"User\" WHERE \"id\" = 'test-user-phase3';"
```

**Output**:
```text
        id        |          email          |      username      |   role    
------------------+-------------------------+--------------------+-----------
 test-user-phase3 | persistence@example.com | persistence_tester | ORGANIZER
(1 row)
```

The test record persisted intact through full container teardown and recreation, confirming reliable volume persistence backed by `postgres_data`.
