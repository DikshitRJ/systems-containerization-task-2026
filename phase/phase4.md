# Phase 4: Network Segmentation and Isolation

## 1. Overview
The goal of Phase 4 is to enforce defense-in-depth network segmentation using Docker bridge networks:
1. **Frontend & Reverse Proxy Network (`frontend-network`)**: Connects the client-facing reverse proxy, the frontend application container, and the backend API container.
2. **Backend & Database Network (`backend-network`)**: A completely isolated internal network connecting strictly the backend API and the PostgreSQL database. Neither the frontend nor the reverse proxy have access to this network.
3. **Host Port Exposure**: Only the reverse proxy publishes ports (`80` and `443`) to the host machine. No internal application or database ports (`4000`, `5432`) are exposed directly to the host.

---

## 2. Target Architecture

```
       [ Host Traffic ]
              │ (Ports 80, 443)
              ▼
   ┌──────────────────────┐
   │     orbis-proxy      │
   └──────────┬───────────┘
              │ (frontend-network)
     ┌────────┴────────┐
     ▼                 ▼
┌──────────────┐  ┌──────────────┐
│orbis-frontend│  │orbis-backend │
└──────────────┘  └──────┬───────┘
                         │ (backend-network - ISOLATED)
                         ▼
                  ┌──────────────┐
                  │   orbis-db   │
                  └──────────────┘
```

* `orbis-proxy`: Member of `frontend-network` only.
* `orbis-frontend`: Member of `frontend-network` only.
* `orbis-backend`: Dual-homed bridge container connected to both `frontend-network` and `backend-network`.
* `orbis-db`: Member of `backend-network` only.

---

## 3. Host-Level Port Isolation Verification

Proving that backend (port 4000) and database (port 5432) are unreachable from the host machine:

### Command
```bash
nc -zv -w 2 127.0.0.1 4000
nc -zv -w 2 127.0.0.1 5432
```

### Raw Terminal Output
```text
nc: connect to 127.0.0.1 port 4000 (tcp) failed: Connection refused
nc: connect to 127.0.0.1 port 5432 (tcp) failed: Connection refused
```
No ports other than the reverse proxy's ports are reachable from the host.

---

## 4. Container-Level Network Isolation Verification

### Test 1: Frontend Container Attempting to Reach Database
The frontend container should have zero visibility or route to `database:5432`.

```bash
docker compose exec frontend nc -zv -w 2 database 5432
```

**Output**:
```text
nc: bad address 'database'
```
DNS resolution and TCP routing to the database fail completely from the frontend tier.

---

### Test 2: Proxy Container Attempting to Reach Database
The reverse proxy should have zero direct visibility to the database container.

```bash
docker compose exec proxy nc -zv -w 2 database 5432
```

**Output**:
```text
nc: bad address 'database'
```
The reverse proxy cannot resolve or communicate directly with `database`.

---

### Test 3: Backend Container Communicating with Database
The backend is an authorized member of `backend-network` and connects directly to `database:5432`.

```bash
docker compose exec backend nc -zv -w 2 database 5432
```

**Output**:
```text
database (172.21.0.2:5432) open
```
Internal communication between backend and database is active and healthy.

---

## 5. Docker Network Inspection Output

### Frontend Network (`docker network inspect systems-containerization-task-2026_frontend-network`)
```json
[
    {
        "Name": "systems-containerization-task-2026_frontend-network",
        "Id": "37bf4a6915b3d49034914f78fc359f844eff44a744aa270764987920b00c6edf",
        "Created": "2026-10-19T17:10:20.953475565+05:30",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {},
        "Labels": {
            "com.docker.compose.config-hash": "28f19db97f83dc8a5f7752184cae2632ea7c2c5aa3c770a67eba1a215f5b3d62",
            "com.docker.compose.network": "frontend-network",
            "com.docker.compose.project": "systems-containerization-task-2026",
            "com.docker.compose.version": "5.4.0"
        },
        "Containers": {
            "6a23945e0cd76a38cf2ac457958c6a416a33508181770a5d136c48645cd2d1a7": {
                "Name": "orbis-proxy",
                "EndpointID": "4eaa48b6b7fdc431e187565ff66ea4509d60a3667f17f1d246179481174fd0f8",
                "MacAddress": "a2:f3:4e:36:19:9d",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
            "d9f6a06abecf274adb30cb51e218ab46da55bf79418ae7f3dd6f4e3f8b034368": {
                "Name": "orbis-frontend",
                "EndpointID": "a3bc07dfaf3d7d9d4552bb89fa3230e081f6f285982574fe86de9509ff396d1c",
                "MacAddress": "c6:73:96:b9:07:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "e2367d04bddb3c75d965bfe492f96c9f5fbc08c928ff5cd2576e7fe8c9f92b6c": {
                "Name": "orbis-backend",
                "EndpointID": "3905c1e5c0cad1aada877bc8f5c4372728a1b040c1755a9652f21c1b0af8e6cd",
                "MacAddress": "fa:ca:c2:62:90:df",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.18.0.0/16": {
                        "IPsInUse": 6,
                        "DynamicIPsAvailable": 65530
                    }
                }
            }
        }
    }
]
```

---

### Backend Network (`docker network inspect systems-containerization-task-2026_backend-network`)
```json
[
    {
        "Name": "systems-containerization-task-2026_backend-network",
        "Id": "17ff998c6e8baaa85383e5070858befe264a419aaebf56ccd7136a8ddfdb92f6",
        "Created": "2026-10-19T17:10:20.952903018+05:30",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {},
        "Labels": {
            "com.docker.compose.config-hash": "f676798fdc08f6419e94b6b99adcd474105630dc6bff59edc2dd4d3c522acdf7",
            "com.docker.compose.network": "backend-network",
            "com.docker.compose.project": "systems-containerization-task-2026",
            "com.docker.compose.version": "5.4.0"
        },
        "Containers": {
            "e2367d04bddb3c75d965bfe492f96c9f5fbc08c928ff5cd2576e7fe8c9f92b6c": {
                "Name": "orbis-backend",
                "EndpointID": "83fa9f620e4bf16be28a9b8272bbec3716048cce25fb4ae45f17bdf8d35daeaa",
                "MacAddress": "02:b3:9f:dc:5f:a1",
                "IPv4Address": "172.21.0.3/16",
                "IPv6Address": ""
            },
            "f620d5f991143452b9fc7369324a29d69dad9e784b64074f465673089780c4a3": {
                "Name": "orbis-db",
                "EndpointID": "af847c55c387f03ebc325d4162adeca2c1aa2eb04aae87c9dd8e17e97ffb0958",
                "MacAddress": "ca:85:21:56:a5:58",
                "IPv4Address": "172.21.0.2/16",
                "IPv6Address": ""
            }
        },
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.21.0.0/16": {
                        "IPsInUse": 5,
                        "DynamicIPsAvailable": 65531
                    }
                }
            }
        }
    }
]
```
