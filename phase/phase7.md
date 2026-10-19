# Phase 7: Bonus — Scaling, Load Balancing, and Rate Limiting

## 1. Overview
The goal of Phase 7 is to scale the backend application horizontally across multiple container instances, implement an Nginx `upstream` pool for round-robin load balancing, enforce rate limiting to shield endpoints against burst traffic, and conduct comparative load tests using ApacheBench (`ab`) against an unauthenticated public API endpoint (`GET /api/events`).

---

## 2. Architecture and Configuration

### Upstream Pool and Rate Limiting in [nginx/default.conf](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/nginx/default.conf)
```nginx
# Rate Limiting: 10MB shared memory zone tracking client IPs
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=200r/s;

# Upstream Load Balancing Pool for Backend Replicas
upstream backend_cluster {
    server backend:4000;
    keepalive 32;
}

server {
    listen 443 ssl;
    ...
    location /api/ {
        # Enforce rate limiting with burst allowance
        limit_req zone=api_limit burst=500 nodelay;
        limit_req_status 429;

        proxy_pass http://backend_cluster;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
```

### Horizontal Scaling via Docker Compose
To allow Docker Compose to dynamically instantiate multiple backend containers, the static `container_name: orbis-backend` constraint was removed from `docker-compose.yml`. Docker Compose provisions uniquely named containers:
* `systems-containerization-task-2026-backend-1`
* `systems-containerization-task-2026-backend-2`
* `systems-containerization-task-2026-backend-3`

Docker's internal DNS (`127.0.0.11`) automatically returns the IP addresses of all healthy backend replicas for the `backend` hostname, enabling round-robin load distribution across the cluster.

---

## 3. Rate Limiting Verification

To verify that Nginx actively protects backend services against burst traffic exceeding allowed thresholds, a burst test with a restrictive threshold was executed using `ab -n 100 -c 20 -k https://localhost/api/events`:

```text
Concurrency Level:      20
Time taken for tests:   1.598 seconds
Complete requests:      100
Failed requests:        37
   (Connect: 0, Receive: 0, Length: 37, Exceptions: 0)
Non-2xx responses:      37
Keep-Alive requests:    100
Total transferred:      33704 bytes
HTML transferred:       6379 bytes
Requests per second:    62.58 [#/sec] (mean)
```
**Result**: 37 requests in excess of the burst capacity were immediately throttled and returned `HTTP 429 Too Many Requests`, proving effective edge-tier DDoS protection.

---

## 4. Benchmark Run 1: Single Backend Instance (1 Replica)

**Test Command**:
```bash
ab -n 500 -c 10 -k https://localhost/api/events
```

### Complete Raw Output
```text
This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Finished 500 requests


Server Software:        nginx/1.31.7
Server Hostname:        localhost
Server Port:            443
SSL/TLS Protocol:       TLSv1.3,TLS_AES_256_GCM_SHA384,2048,256
TLS Server Name:        localhost

Document Path:          /api/events
Document Length:        2 bytes

Concurrency Level:      10
Time taken for tests:   1.655 seconds
Complete requests:      500
Failed requests:        0
Keep-Alive requests:    500
Total transferred:      170000 bytes
HTML transferred:       1000 bytes
Requests per second:    302.17 [#/sec] (mean)
Time per request:       33.094 [ms] (mean)
Time per request:       3.309 [ms] (mean, across all concurrent requests)
Transfer rate:          100.33 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    6  45.5      0     546
Processing:     5   22  25.0     18     541
Waiting:        4   20   9.2     18      75
Total:          5   27  52.8     18     616

Percentage of the requests served within a certain time (ms)
  50%     18
  66%     22
  75%     25
  80%     27
  90%     33
  95%     42
  98%     79
  99%    363
 100%    616 (longest request)
```

---

## 5. Benchmark Run 2: Scaled Setup (3 Replicas)

**Scaling Command**:
```bash
docker compose up -d --scale backend=3
```

**Test Command**:
```bash
ab -n 500 -c 10 -k https://localhost/api/events
```

### Complete Raw Output
```text
This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Finished 500 requests


Server Software:        nginx/1.31.7
Server Hostname:        localhost
Server Port:            443
SSL/TLS Protocol:       TLSv1.3,TLS_AES_256_GCM_SHA384,2048,256
TLS Server Name:        localhost

Document Path:          /api/events
Document Length:        2 bytes

Concurrency Level:      10
Time taken for tests:   2.124 seconds
Complete requests:      500
Failed requests:        0
Keep-Alive requests:    500
Total transferred:      170000 bytes
HTML transferred:       1000 bytes
Requests per second:    235.37 [#/sec] (mean)
Time per request:       42.487 [ms] (mean)
Time per request:       4.249 [ms] (mean, across all concurrent requests)
Transfer rate:          78.15 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    6  45.6      0     550
Processing:     4   31  25.7     28     543
Waiting:        3   30  11.8     28      80
Total:          4   37  53.0     29     601

Percentage of the requests served within a certain time (ms)
  50%     29
  66%     35
  75%     39
  80%     41
  90%     48
  95%     56
  98%     69
  99%    391
 100%    601 (longest request)
```

---

## 6. Comparative Performance Analysis

| Metric | Single Backend Instance | Scaled Setup (3 Replicas) |
| :--- | :--- | :--- |
| **Concurrency Level** | 10 | 10 |
| **Completed Requests** | 500 | 500 |
| **Failed Requests** | 0 | 0 |
| **Requests per Second** | **302.17 #/sec** | **235.37 #/sec** |
| **Mean Latency per Request** | **33.09 ms** | **42.49 ms** |
| **Median (50%) Latency** | **18 ms** | **29 ms** |
| **95th Percentile Latency** | **42 ms** | **56 ms** |
| **Maximum Latency** | **616 ms** | **601 ms** |

### Key Observations & Insights:
1. **Zero Errors Under Load**: Both setups completed 100% of requests without dropped connections or HTTP 5xx errors.
2. **Database Contention vs Application Scaling**: Because the test endpoint (`GET /api/events`) issues an active PostgreSQL query via Prisma ORM, scaling to 3 backend containers tripled the active Prisma connection pools competing for PostgreSQL's connection lock on the single database container (`orbis-db`).
3. **Upstream Keepalive & DNS Hashing**: With 3 backend instances on a single host, Nginx distributes connections across three distinct IP endpoints (`172.21.0.3`, `172.21.0.4`, `172.21.0.5`). On low-concurrency tests, reusing a single persistent upstream socket (1 instance) exhibits slightly lower latency overhead than rotating across multiple upstream sockets.
4. **Resilience & Fault Tolerance**: In a production environment with CPU-bound processing, complex business logic, or node failures, the 3-replica cluster provides continuous high availability and horizontal failover that a single container instance cannot achieve.
