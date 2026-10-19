# Phase 5: Nginx SSL Termination and HTTPS Redirection

## 1. Overview
The goal of Phase 5 is to configure secure HTTPS communication for the multi-container stack. This entails:
1. Generating an SSL/TLS certificate and private key.
2. Configuring Nginx to terminate SSL on port 443 and proxy decrypted traffic to internal services (`frontend` and `backend`).
3. Automatically redirecting all unencrypted HTTP traffic arriving on port 80 to HTTPS with an HTTP 301 Moved Permanently status code.

---

## 2. Certificate Generation

A 2048-bit RSA private key and self-signed X.509 certificate were generated using OpenSSL with Subject Alternative Names (SAN) for `localhost` and `127.0.0.1`:

```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/CN=localhost/O=Orbis/C=US" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

The resulting certificate files are stored in `nginx/certs/`:
* `nginx/certs/server.crt`: Public X.509 certificate.
* `nginx/certs/server.key`: Private RSA key.

---

## 3. Nginx Configuration

The updated [nginx/default.conf](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/nginx/default.conf) defines two server blocks:

```nginx
# HTTP Server: Redirect all traffic to HTTPS
server {
    listen 80;
    server_name localhost;

    # Automatically redirect all incoming HTTP traffic to HTTPS
    return 301 https://$host$request_uri;
}

# HTTPS Server: Terminate SSL and proxy traffic
server {
    listen 443 ssl;
    server_name localhost;

    # SSL Certificate and Key Configuration
    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    # Secure SSL Protocols and Ciphers
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Docker internal DNS resolver
    resolver 127.0.0.11 valid=10s ipv6=off;

    location /api/ {
        set $backend_service http://backend:4000;
        proxy_pass $backend_service;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location / {
        set $frontend_service http://frontend:80;
        proxy_pass $frontend_service;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

In [docker-compose.yml](file:///mnt/Data/Coding/WECRecs/systems/containerization/systems-containerization-task-2026/docker-compose.yml), port `443:443` was exposed and the certificate directory was mounted into the proxy container:
```yaml
  proxy:
    image: nginx:alpine
    container_name: orbis-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro,z
      - ./nginx/certs:/etc/nginx/certs:ro,z
```

---

## 4. Verification and Testing

### 1. HTTP to HTTPS Redirection Verification (Port 80 -> Port 443)
Sending requests over plaintext HTTP returns an immediate 301 redirect to HTTPS for both root and API endpoints:

```bash
curl -I http://localhost/
curl -I http://localhost/api/events
```

**Output**:
```text
HTTP/1.1 301 Moved Permanently
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:43:35 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: https://localhost/

HTTP/1.1 301 Moved Permanently
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:43:35 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: https://localhost/api/events
```

---

### 2. HTTPS Verification on Port 443

#### Frontend HTTPS Request:
```bash
curl -k -I https://localhost/
```

**Output**:
```text
HTTP/1.1 200 OK
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:43:40 GMT
Content-Type: text/html
Content-Length: 452
Connection: keep-alive
Last-Modified: Mon, 19 Oct 2026 11:38:50 GMT
ETag: "6ad6014a-1c4"
Accept-Ranges: bytes
```

#### Backend API HTTPS Request:
```bash
curl -k -i https://localhost/api/events
```

**Output**:
```text
HTTP/1.1 200 OK
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 11:43:40 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 2
Connection: keep-alive
X-Powered-By: Express
Access-Control-Allow-Origin: http://localhost:3000
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"2-l9Fw4VUO7kr8CvBlt4zaMCqXZ0w"

[]
```

---

### 3. TLS Handshake Details
Inspecting the cryptographic handshake:
```bash
curl -vk https://localhost/ 2>&1 | grep -E 'SSL connection|server certificate|subject|issuer'
```

**Output**:
```text
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519MLKEM768 / RSASSA-PSS
*   subject: CN=localhost; O=Orbis; C=US
*   issuer: CN=localhost; O=Orbis; C=US
```
Nginx successfully establishes a modern TLS 1.3 encrypted tunnel, terminates SSL at the edge, and proxies decrypted traffic to the backend and frontend services.
