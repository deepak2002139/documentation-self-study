# Nginx - Complete Guide with Integration Testing

## Table of Contents
1. [Introduction to Nginx](#introduction-to-nginx)
2. [Nginx Architecture](#nginx-architecture)
3. [Core Concepts](#core-concepts)
4. [Nginx as Web Server](#nginx-as-web-server)
5. [Nginx as Reverse Proxy](#nginx-as-reverse-proxy)
6. [Nginx Configuration Deep Dive](#nginx-configuration-deep-dive)
7. [Integration Testing with Nginx](#integration-testing-with-nginx)
8. [Communication Flow Diagrams](#communication-flow-diagrams)
9. [Practical Implementation](#practical-implementation)
10. [Testing Scenarios](#testing-scenarios)
11. [Best Practices](#best-practices)
12. [Troubleshooting](#troubleshooting)

---

## Introduction to Nginx

### What is Nginx?

**Nginx** (pronounced "engine-x") is a high-performance, open-source web server, reverse proxy server, load balancer, mail proxy server, and HTTP cache.

### Key Features

1. **High Performance**: Can handle 10,000+ concurrent connections
2. **Low Memory Footprint**: Uses ~2.5 MB per 10,000 connections
3. **Event-Driven Architecture**: Asynchronous, non-blocking
4. **Reverse Proxy**: Routes requests to backend servers
5. **Load Balancing**: Distributes traffic across multiple servers
6. **SSL/TLS Termination**: Handles HTTPS encryption/decryption
7. **Static Content Serving**: Efficiently serves static files
8. **Caching**: HTTP response caching
9. **Rate Limiting**: Controls request rates
10. **URL Rewriting**: Modifies URLs on the fly

### Why Nginx?

```
Traditional Web Server (Apache)     vs     Nginx
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Thread-based (one thread per           Event-driven (single thread
connection)                            handles multiple connections)

High memory usage under load           Low memory usage

Context switching overhead             Minimal context switching

Good for dynamic content               Excellent for static content
                                      and reverse proxy

Max ~1000 concurrent connections       Max 10,000+ concurrent connections
```

---

## Nginx Architecture

### Process Model

```
┌─────────────────────────────────────────────────────────┐
│                    Nginx Process Model                   │
├─────────────────────────────────────────────────────────┤
│                                                           │
│                    Master Process                         │
│         (reads config, manages workers)                   │
│                         │                                 │
│         ┌───────────────┼───────────────┐                │
│         ▼               ▼               ▼                 │
│    Worker Process  Worker Process  Worker Process        │
│    (handles       (handles        (handles               │
│     requests)      requests)       requests)              │
│         │               │               │                 │
│         └───────────────┴───────────────┘                │
│                         │                                 │
│                         ▼                                 │
│              Event Loop (epoll/kqueue)                    │
│         (monitors sockets, handles events)                │
│                         │                                 │
│         ┌───────────────┼───────────────┐                │
│         ▼               ▼               ▼                 │
│    Connection 1    Connection 2    Connection N          │
│    (non-blocking) (non-blocking)  (non-blocking)         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Event-Driven Architecture

```java
// Pseudo-code representation of Nginx event loop

class NginxWorkerProcess {
    
    EventQueue eventQueue;
    ConnectionPool connections;
    
    void eventLoop() {
        while (true) {
            // Wait for events (non-blocking)
            Event[] events = eventQueue.wait(timeout);
            
            for (Event event : events) {
                if (event.isReadable()) {
                    handleRead(event.connection);
                } else if (event.isWritable()) {
                    handleWrite(event.connection);
                } else if (event.isNewConnection()) {
                    acceptConnection(event);
                }
            }
            
            // Process timeouts
            handleTimeouts();
        }
    }
    
    void handleRead(Connection conn) {
        // Read data without blocking
        byte[] data = conn.readNonBlocking();
        
        // Process request
        Request request = parseRequest(data);
        Response response = processRequest(request);
        
        // Queue response for writing
        conn.queueWrite(response);
    }
}
```

### Key Components

1. **Master Process**
   - Reads and validates configuration
   - Creates and manages worker processes
   - Handles signals (reload, restart, shutdown)
   - Manages cache

2. **Worker Processes**
   - Handle actual client requests
   - Process events (connections, reads, writes)
   - Execute Nginx modules
   - Number typically equals CPU cores

3. **Cache Manager**
   - Manages disk cache
   - Cleans up expired cache entries

4. **Cache Loader**
   - Loads cache metadata on startup
   - Validates cache integrity

---

## Core Concepts

### 1. Contexts (Configuration Blocks)

```nginx
# Main Context (global settings)
user nginx;
worker_processes auto;

# Events Context
events {
    worker_connections 1024;
}

# HTTP Context
http {
    # Server Context
    server {
        listen 80;
        server_name example.com;
        
        # Location Context
        location / {
            # Directives here
        }
        
        location /api {
            # Directives here
        }
    }
    
    # Another Server Context
    server {
        listen 443 ssl;
        server_name secure.example.com;
    }
}
```

### 2. Directives

```nginx
# Simple Directive (name value;)
worker_processes 4;

# Block Directive (name { ... })
events {
    worker_connections 1024;
}

# Array Directive
server_name example.com www.example.com;
```

### 3. Variables

```nginx
# Built-in Variables
$host              # Hostname from request
$uri               # Current URI
$request_uri       # Full original request URI
$remote_addr       # Client IP address
$request_method    # HTTP method (GET, POST, etc.)
$scheme            # http or https
$args              # Query string parameters

# Custom Variables
set $my_var "value";
```

### 4. Modules

Nginx functionality is provided by modules:

- **Core Modules**: HTTP, Events, Mail
- **Standard HTTP Modules**: Rewrite, Proxy, FastCGI, SSL
- **Third-party Modules**: Custom functionality

---

## Nginx as Web Server

### Basic Web Server Configuration

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;  # Auto-detect CPU cores
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;  # Efficient event mechanism for Linux
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
    
    access_log /var/log/nginx/access.log main;
    
    # Performance settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Virtual Host Configuration
    include /etc/nginx/conf.d/*.conf;
}
```

### Serving Static Content

```nginx
# /etc/nginx/conf.d/static-site.conf

server {
    listen 80;
    server_name static.example.com;
    
    root /var/www/html;
    index index.html index.htm;
    
    # Main location
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
```

---

## Nginx as Reverse Proxy

### What is a Reverse Proxy?

```
Client Request Flow:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Without Reverse Proxy:
┌────────┐                              ┌────────────┐
│ Client │─────────── Direct ──────────>│   Server   │
└────────┘                              └────────────┘
   (Knows server address)                (Exposed to internet)


With Reverse Proxy:
┌────────┐        ┌───────┐        ┌────────────┐
│ Client │───────>│ Nginx │───────>│   Server   │
└────────┘        └───────┘        └────────────┘
  (Knows only      (Proxy)          (Hidden from client)
   proxy address)
```

### Benefits of Reverse Proxy

1. **Security**: Hides backend server details
2. **Load Balancing**: Distributes traffic across multiple servers
3. **SSL Termination**: Handles HTTPS encryption/decryption
4. **Caching**: Reduces backend load
5. **Compression**: Compresses responses
6. **Rate Limiting**: Protects backend from overload
7. **Request Routing**: Routes based on URL, headers, etc.

### Basic Reverse Proxy Configuration

```nginx
# /etc/nginx/conf.d/reverse-proxy.conf

upstream backend_servers {
    # Define backend servers
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

server {
    listen 80;
    server_name api.example.com;
    
    # Proxy all requests to backend
    location / {
        proxy_pass http://backend_servers;
        
        # Proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Proxy timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # Error handling
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
    }
}
```

### Load Balancing Algorithms

```nginx
# 1. Round Robin (default)
upstream backend {
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

# 2. Least Connections
upstream backend {
    least_conn;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

# 3. IP Hash (session persistence)
upstream backend {
    ip_hash;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

# 4. Weighted Round Robin
upstream backend {
    server server1.example.com weight=3;
    server server2.example.com weight=2;
    server server3.example.com weight=1;
}

# 5. Generic Hash
upstream backend {
    hash $request_uri consistent;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

# 6. Health Checks
upstream backend {
    server server1.example.com max_fails=3 fail_timeout=30s;
    server server2.example.com max_fails=3 fail_timeout=30s;
    server server3.example.com backup;  # Backup server
}
```

---

## Nginx Configuration Deep Dive

### Complete Enterprise Configuration

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;  # Maximum open files per worker

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Basic Settings
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    charset utf-8;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;  # Hide Nginx version
    
    # Timeouts
    client_body_timeout 12s;
    client_header_timeout 12s;
    keepalive_timeout 65s;
    send_timeout 10s;
    
    # Buffer Sizes
    client_body_buffer_size 10K;
    client_header_buffer_size 1k;
    client_max_body_size 8m;
    large_client_header_buffers 2 1k;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main;
    
    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 256;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;
    
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    
    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # Proxy Cache
    proxy_cache_path /var/cache/nginx/proxy 
                     levels=1:2 
                     keys_zone=proxy_cache:10m 
                     max_size=1g 
                     inactive=60m 
                     use_temp_path=off;
    
    # Include virtual host configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Advanced Reverse Proxy Configuration

```nginx
# /etc/nginx/conf.d/advanced-proxy.conf

# Upstream configuration with health checks
upstream app_backend {
    least_conn;
    
    # Backend servers
    server app1.internal.com:8080 max_fails=3 fail_timeout=30s weight=3;
    server app2.internal.com:8080 max_fails=3 fail_timeout=30s weight=2;
    server app3.internal.com:8080 max_fails=3 fail_timeout=30s weight=1;
    
    # Backup server (used when all primary servers are down)
    server backup.internal.com:8080 backup;
    
    # Keepalive connections to backend
    keepalive 32;
    keepalive_timeout 60s;
}

# Microservices upstreams
upstream user_service {
    server user-svc:8080;
}

upstream order_service {
    server order-svc:8080;
}

upstream payment_service {
    server payment-svc:8080;
}

# Main server block
server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name api.example.com;
    
    # SSL certificates
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # Logging
    access_log /var/log/nginx/api-access.log main;
    error_log /var/log/nginx/api-error.log warn;
    
    # Rate limiting
    limit_req zone=api burst=20 nodelay;
    limit_conn addr 10;
    
    # Client body size
    client_max_body_size 10M;
    
    # CORS Headers
    add_header Access-Control-Allow-Origin "*" always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
    
    # Handle OPTIONS preflight
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Static content
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # API Gateway - User Service
    location /api/users {
        # Rate limit for this endpoint
        limit_req zone=api burst=10 nodelay;
        
        # Proxy to user service
        proxy_pass http://user_service;
        
        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID $request_id;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
        
        # Error handling
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_intercept_errors on;
        
        # Caching
        proxy_cache proxy_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;
        
        add_header X-Cache-Status $upstream_cache_status;
    }
    
    # API Gateway - Order Service
    location /api/orders {
        proxy_pass http://order_service;
        include /etc/nginx/proxy_params;
    }
    
    # API Gateway - Payment Service
    location /api/payments {
        proxy_pass http://payment_service;
        include /etc/nginx/proxy_params;
        
        # Higher timeout for payment processing
        proxy_read_timeout 120s;
    }
    
    # Catch-all - Main Application
    location / {
        proxy_pass http://app_backend;
        include /etc/nginx/proxy_params;
    }
    
    # Custom error pages
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    
    location = /404.html {
        root /var/www/error-pages;
        internal;
    }
    
    location = /50x.html {
        root /var/www/error-pages;
        internal;
    }
}
```

### Reusable Proxy Parameters

```nginx
# /etc/nginx/proxy_params

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Request-ID $request_id;

proxy_http_version 1.1;
proxy_set_header Connection "";

proxy_connect_timeout 5s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 4k;

proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
```

---

## Integration Testing with Nginx

### Why Test with Nginx?

Integration testing with Nginx ensures:
1. **Realistic Environment**: Tests mimic production setup
2. **Proxy Behavior**: Validates header forwarding, timeouts
3. **Load Balancing**: Tests multiple backend instances
4. **SSL/TLS**: Validates HTTPS configuration
5. **Rate Limiting**: Tests throttling behavior
6. **Caching**: Validates cache behavior
7. **Error Handling**: Tests failover scenarios

### Testing Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│              Integration Test Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────┐           │
│  │           Test Execution Environment          │           │
│  │              (JUnit/TestNG)                   │           │
│  └──────────────┬───────────────────────────────┘           │
│                 │                                             │
│                 │ 1. Start Containers                         │
│                 ▼                                             │
│  ┌──────────────────────────────────────────────┐           │
│  │        Docker Compose / Testcontainers        │           │
│  │                                               │           │
│  │  ┌─────────────┐        ┌─────────────┐     │           │
│  │  │   Nginx     │        │  Spring     │     │           │
│  │  │  Container  │───────>│  Boot App   │     │           │
│  │  │   (Port     │        │  Container  │     │           │
│  │  │    8080)    │        │ (Port 8080) │     │           │
│  │  └─────────────┘        └─────────────┘     │           │
│  │                                               │           │
│  └──────────────────────────────────────────────┘           │
│                 │                                             │
│                 │ 2. Send HTTP Requests                       │
│                 │ 3. Receive Responses                        │
│                 │ 4. Assert Results                           │
│                 ▼                                             │
│  ┌──────────────────────────────────────────────┐           │
│  │           Test Assertions                     │           │
│  │  - Status codes                               │           │
│  │  - Response headers                           │           │
│  │  - Response body                              │           │
│  │  - Response times                             │           │
│  └──────────────────────────────────────────────┘           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Communication Flow Diagrams

### Detailed Request-Response Flow

```
┌────────────────────────────────────────────────────────────────────┐
│         Complete Request-Response Flow in Integration Test         │
└────────────────────────────────────────────────────────────────────┘

Step 1: Test Initiation
━━━━━━━━━━━━━━━━━━━━━━
┌─────────────┐
│  Test Class │
│  (JUnit 5)  │
└──────┬──────┘
       │
       │ @BeforeAll: Start containers
       ▼
┌─────────────────────┐
│  Testcontainers     │
│  Framework          │
└──────┬──────────────┘
       │
       ├─────────────────────────────────────┐
       │                                     │
       │ Start Container 1                   │ Start Container 2
       ▼                                     ▼
┌──────────────────┐              ┌──────────────────┐
│ Nginx Container  │              │  App Container   │
│ Port: 8080       │              │  Port: 8080      │
│ Network: test-net│              │  Network: test-net│
└──────────────────┘              └──────────────────┘


Step 2: Test Execution
━━━━━━━━━━━━━━━━━━━━━
┌─────────────┐
│ Test Method │
└──────┬──────┘
       │
       │ RestTemplate.getForEntity("http://localhost:8080/api/users")
       ▼
┌──────────────────────────────────────────────────────────────┐
│                    HTTP Request Packet                        │
├──────────────────────────────────────────────────────────────┤
│ GET /api/users HTTP/1.1                                      │
│ Host: localhost:8080                                         │
│ User-Agent: Java/17 HttpClient                               │
│ Accept: application/json                                     │
│ Connection: keep-alive                                       │
└──────────────────────────────────────────────────────────────┘
       │
       │ TCP Connection
       ▼


Step 3: Nginx Receives Request
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌──────────────────────────────────────────────────────────────┐
│              Nginx Container (Port 8080)                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Accept TCP Connection                                    │
│     - epoll notifies worker of new connection                │
│     - Worker accepts connection (non-blocking)               │
│                                                               │
│  2. Parse HTTP Request                                       │
│     - Read request line: GET /api/users HTTP/1.1             │
│     - Read headers                                           │
│     - Validate request                                       │
│                                                               │
│  3. Match Location Block                                     │
│     - Check: location /api/users { ... }                     │
│     - Apply directives from matched location                 │
│                                                               │
│  4. Apply Proxy Configuration                                │
│     - proxy_pass http://app_backend;                         │
│     - Add proxy headers:                                     │
│       * X-Real-IP: 172.18.0.1                               │
│       * X-Forwarded-For: 172.18.0.1                         │
│       * X-Forwarded-Proto: http                             │
│       * Host: localhost:8080                                │
│                                                               │
│  5. Select Backend Server                                    │
│     - Use load balancing algorithm (round-robin)             │
│     - Check backend health                                   │
│     - Selected: app_backend (app:8080)                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
       │
       │ Forward Request
       ▼


Step 4: Request Forwarding to Backend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌──────────────────────────────────────────────────────────────┐
│              Modified HTTP Request to Backend                 │
├──────────────────────────────────────────────────────────────┤
│ GET /api/users HTTP/1.1                                      │
│ Host: localhost:8080                                         │
│ X-Real-IP: 172.18.0.1                                       │
│ X-Forwarded-For: 172.18.0.1                                 │
│ X-Forwarded-Proto: http                                     │
│ X-Request-ID: 550e8400-e29b-41d4-a716-446655440000          │
│ User-Agent: Java/17 HttpClient                               │
│ Accept: application/json                                     │
│ Connection: keep-alive                                       │
└──────────────────────────────────────────────────────────────┘
       │
       │ TCP Connection (Docker network: test-net)
       ▼


Step 5: Spring Boot App Processes Request
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌──────────────────────────────────────────────────────────────┐
│         Spring Boot Application Container                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Tomcat Receives Request                                  │
│     - HTTP Connector (Port 8080)                             │
│     - Creates HttpServletRequest object                      │
│                                                               │
│  2. DispatcherServlet Processing                             │
│     - Request mapping: GET /api/users                        │
│     - Find controller method                                 │
│                                                               │
│  3. Controller Execution                                     │
│     @GetMapping("/api/users")                                │
│     public List<User> getUsers() {                           │
│         return userService.getAllUsers();                    │
│     }                                                         │
│                                                               │
│  4. Service Layer                                            │
│     - Business logic execution                               │
│     - Database query (if needed)                             │
│     - Data transformation                                    │
│                                                               │
│  5. Response Generation                                      │
│     - Serialize data to JSON                                 │
│     - Set response headers                                   │
│     - Set status code: 200 OK                                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
       │
       │ Generate Response
       ▼


Step 6: Spring Boot Sends Response
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌──────────────────────────────────────────────────────────────┐
│              HTTP Response from Spring Boot                   │
├──────────────────────────────────────────────────────────────┤
│ HTTP/1.1 200 OK                                              │
│ Content-Type: application/json;charset=UTF-8                 │
│ Transfer-Encoding: chunked                                   │
│ Date: Fri, 17 Oct 2025 13:42:02 GMT                         │
│                                                               │
│ {                                                             │
│   "users": [                                                  │
│     {"id": 1, "name": "Deepak Kumar", "email": "..."},      │
│     {"id": 2, "name": "John Doe", "email": "..."}           │
│   ]                                                           │
│ }                                                             │
└──────────────────────────────────────────────────────────────┘
       │
       │ Send back through TCP connection
       ▼


Step 7: Nginx Receives Backend Response
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌──────────────────────────────────────────────────────────────┐
│              Nginx Processing Backend Response                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Receive Response from Backend                            │
│     - Read response headers                                  │
│     - Read response body                                     │
│                                                               │
│  2. Apply Response Processing                                │
│     - Gzip compression (if enabled and applicable)           │
│     - Add/modify headers:                                    │
│       * X-Cache-Status: MISS                                │
│       * Server: nginx/1.21.6                                │
│                                                               │
│  3. Caching Decision                                         │
│     - Check cache configuration                              │
│     - Store in cache if cacheable                            │
│     - Cache key: GET/api/users                               │
│                                                               │
│  4. Response Buffering                                       │
│     - Buffer response in memory                              │
│     - Apply proxy_buffer settings                            │
│                                                               │
│  5. Log Request                                              │
│     - Access log entry                                       │
│     - Request time: 45ms                                     │
│     - Upstream time: 42ms                                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
       │
       │ Forward Response
       ▼


Step 8: Nginx Sends Response to Client
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌──────────────────────────────────────────────────────────────┐
│              Final HTTP Response to Test Client               │
├──────────────────────────────────────────────────────────────┤
│ HTTP/1.1 200 OK                                              │
│ Server: nginx/1.21.6                                         │
│ Content-Type: application/json;charset=UTF-8                 │
│ Transfer-Encoding: chunked                                   │
│ Connection: keep-alive                                       │
│ X-Cache-Status: MISS                                         │
│ Date: Fri, 17 Oct 2025 13:42:02 GMT                         │
│                                                               │
│ {                                                             │
│   "users": [                                                  │
│     {"id": 1, "name": "Deepak Kumar", "email": "..."},      │
│     {"id": 2, "name": "John Doe", "email": "..."}           │
│   ]                                                           │
│ }                                                             │
└──────────────────────────────────────────────────────────────┘
       │
       │ TCP Response
       ▼


Step 9: Test Validates Response
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌─────────────┐
│ Test Method │
└──────┬──────┘
       │
       │ ResponseEntity<String> response = ...
       ▼
┌──────────────────────────────────────────────────────────────┐
│                    Assertion Checks                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Status Code Assertion                                    │
│     assertEquals(200, response.getStatusCode().value());     │
│     ✓ PASS                                                   │
│                                                               │
│  2. Content-Type Assertion                                   │
│     assertEquals("application/json",                         │
│                  response.getHeaders().getContentType());    │
│     ✓ PASS                                                   │
│                                                               │
│  3. Response Body Assertion                                  │
│     assertTrue(response.getBody().contains("Deepak Kumar")); │
│     ✓ PASS                                                   │
│                                                               │
│  4. Response Time Assertion                                  │
│     assertTrue(responseTime < 1000); // < 1 second           │
│     ✓ PASS (45ms)                                           │
│                                                               │
│  5. Custom Header Assertion                                  │
│     assertNotNull(response.getHeaders().get("X-Cache-Status"));│
│     ✓ PASS                                                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
   Test PASSED ✓
```

### Sequence Diagram

```
Test Client    Testcontainers    Nginx Container    Spring Boot App
    │                │                  │                  │
    │  startAll()    │                  │                  │
    ├───────────────>│                  │                  │
    │                │  docker run nginx│                  │
    │                ├─────────────────>│                  │
    │                │                  │                  │
    │                │  docker run app  │                  │
    │                ├─────────────────────────────────────>│
    │                │                  │                  │
    │                │  wait for ready  │                  │
    │                │<─────────────────│                  │
    │                │                  │                  │
    │                │  wait for ready  │                  │
    │                │<────────────────────────────────────│
    │                │                  │                  │
    │  containers ready                 │                  │
    │<───────────────│                  │                  │
    │                │                  │                  │
    │  GET /api/users                   │                  │
    ├──────────────────────────────────>│                  │
    │                │                  │                  │
    │                │                  │  proxy_pass      │
    │                │                  ├─────────────────>│
    │                │                  │                  │
    │                │                  │  process request │
    │                │                  │  ──┐             │
    │                │                  │    │ execute     │
    │                │                  │  <─┘ controller  │
    │                │                  │                  │
    │                │                  │  HTTP 200 + JSON │
    │                │                  │<─────────────────│
    │                │                  │                  │
    │                │                  │  add headers     │
    │                │                  │  cache (optional)│
    │                │                  │  ──┐             │
    │                │                  │  <─┘             │
    │                │                  │                  │
    │  HTTP 200 + JSON + Nginx headers  │                  │
    │<──────────────────────────────────│                  │
    │                │                  │                  │
    │  assert response                  │                  │
    │  ──┐           │                  │                  │
    │    │ verify    │                  │                  │
    │  <─┘ status    │                  │                  │
    │                │                  │                  │
    │  test passed ✓ │                  │                  │
    │                │                  │                  │
    │  @AfterAll     │                  │                  │
    │  stopAll()     │                  │                  │
    ├───────────────>│                  │                  │
    │                │  docker stop     │                  │
    │                ├─────────────────>│                  │
    │                │                  X                  │
    │                │                                     │
    │                │  docker stop                        │
    │                ├────────────────────────────────────>│
    │                │                                     X
    │                │                                     
    │  cleanup done  │                                     
    │<───────────────│                                     
    │                │                                     
```

---

## Practical Implementation

### Project Structure

```
nginx-integration-test/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/deepak/app/
│   │   │       ├── Application.java
│   │   │       ├── controller/
│   │   │       │   └── UserController.java
│   │   │       ├── service/
│   │   │       │   └── UserService.java
│   │   │       └── model/
│   │   │           └── User.java
│   │   └── resources/
│   │       └── application.properties
│   │
│   └── test/
│       ├── java/
│       │   └── com/deepak/app/
│       │       └── integration/
│       │           ├── NginxIntegrationTest.java
│       │           ├── NginxLoadBalancingTest.java
│       │           ├── NginxCachingTest.java
│       │           └── NginxRateLimitingTest.java
│       │
│       └── resources/
│           ├── nginx/
│           │   └── nginx.conf
│           ├── docker-compose.yml
│           └── application-test.properties
│
├── docker/
│   ├── Dockerfile
│   └── nginx/
│       └── nginx.conf
│
├── pom.xml
└── README.md
```

### Step 1: Maven Dependencies

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.deepak</groupId>
    <artifactId>nginx-integration-test</artifactId>
    <version>1.0.0</version>
    <name>Nginx Integration Test</name>
    <description>Integration testing with Nginx reverse proxy</description>

    <properties>
        <java.version>17</java.version>
        <testcontainers.version>1.19.3</testcontainers.version>
        <rest-assured.version>5.4.0</rest-assured.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starter Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Boot Starter Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Spring Boot Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Testcontainers -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Testcontainers JUnit Jupiter -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Testcontainers Nginx -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>nginx</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- REST Assured for API testing -->
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>${rest-assured.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Awaitility for async testing -->
        <dependency>
            <groupId>org.awaitility</groupId>
            <artifactId>awaitility</artifactId>
            <version>4.2.0</version>
            <scope>test</scope>
        </dependency>

        <!-- Apache HttpClient for testing -->
        <dependency>
            <groupId>org.apache.httpcomponents.client5</groupId>
            <artifactId>httpclient5</artifactId>
            <version>5.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <!-- Maven Surefire for unit tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.2</version>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>
                    </includes>
                </configuration>
            </plugin>

            <!-- Maven Failsafe for integration tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>3.2.2</version>
                <configuration>
                    <includes>
                        <include>**/*IT.java</include>
                        <include>**/integration/**/*Test.java</include>
                    </includes>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### Step 2: Spring Boot Application

```java
package com.deepak.app;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Main Application Class
 */
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
package com.deepak.app.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * User Model
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private String email;
    private String role;
}
```

```java
package com.deepak.app.service;

import com.deepak.app.model.User;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * User Service
 */
@Service
public class UserService {
    
    private final Map<Long, User> users = new ConcurrentHashMap<>();
    private final AtomicLong idCounter = new AtomicLong(1);
    
    public UserService() {
        // Initialize with some data
        createUser(User.builder()
                .name("Deepak Kumar")
                .email("deepak@example.com")
                .role("ADMIN")
                .build());
        
        createUser(User.builder()
                .name("John Doe")
                .email("john@example.com")
                .role("USER")
                .build());
    }
    
    public List<User> getAllUsers() {
        return new ArrayList<>(users.values());
    }
    
    public User getUserById(Long id) {
        return users.get(id);
    }
    
    public User createUser(User user) {
        user.setId(idCounter.getAndIncrement());
        users.put(user.getId(), user);
        return user;
    }
    
    public User updateUser(Long id, User user) {
        if (users.containsKey(id)) {
            user.setId(id);
            users.put(id, user);
            return user;
        }
        return null;
    }
    
    public boolean deleteUser(Long id) {
        return users.remove(id) != null;
    }
}
```

```java
package com.deepak.app.controller;

import com.deepak.app.model.User;
import com.deepak.app.service.UserService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpServletRequest;
import java.util.List;

/**
 * User REST Controller
 */
@Slf4j
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    /**
     * Get all users
     */
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers(HttpServletRequest request) {
        log.info("GET /api/users - Remote Address: {}", request.getRemoteAddr());
        log.info("X-Real-IP: {}", request.getHeader("X-Real-IP"));
        log.info("X-Forwarded-For: {}", request.getHeader("X-Forwarded-For"));
        log.info("X-Forwarded-Proto: {}", request.getHeader("X-Forwarded-Proto"));
        
        List<User> users = userService.getAllUsers();
        return ResponseEntity.ok(users);
    }
    
    /**
     * Get user by ID
     */
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(
            @PathVariable Long id,
            HttpServletRequest request) {
        
        log.info("GET /api/users/{} - Remote Address: {}", id, request.getRemoteAddr());
        
        User user = userService.getUserById(id);
        if (user != null) {
            return ResponseEntity.ok(user);
        }
        return ResponseEntity.notFound().build();
    }
    
    /**
     * Create new user
     */
    @PostMapping
    public ResponseEntity<User> createUser(
            @RequestBody User user,
            HttpServletRequest request) {
        
        log.info("POST /api/users - Remote Address: {}", request.getRemoteAddr());
        log.info("Request Body: {}", user);
        
        User createdUser = userService.createUser(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdUser);
    }
    
    /**
     * Update user
     */
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
            @PathVariable Long id,
            @RequestBody User user,
            HttpServletRequest request) {
        
        log.info("PUT /api/users/{} - Remote Address: {}", id, request.getRemoteAddr());
        
        User updatedUser = userService.updateUser(id, user);
        if (updatedUser != null) {
            return ResponseEntity.ok(updatedUser);
        }
        return ResponseEntity.notFound().build();
    }
    
    /**
     * Delete user
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(
            @PathVariable Long id,
            HttpServletRequest request) {
        
        log.info("DELETE /api/users/{} - Remote Address: {}", id, request.getRemoteAddr());
        
        boolean deleted = userService.deleteUser(id);
        if (deleted) {
            return ResponseEntity.noContent().build();
        }
        return ResponseEntity.notFound().build();
    }
    
    /**
     * Health check endpoint
     */
    @GetMapping("/health")
    public ResponseEntity<String> health() {
        return ResponseEntity.ok("OK");
    }
}
```

### Step 3: Nginx Configuration for Testing

```nginx
# src/test/resources/nginx/nginx.conf

user nginx;
worker_processes 1;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    # Upstream backend servers
    upstream app_backend {
        # Spring Boot app container
        # This will be replaced with actual container name in tests
        server app:8080 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name localhost;

        # Proxy settings
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Health check
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Proxy to backend
        location /api/ {
            proxy_pass http://app_backend;
            
            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            
            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
            
            # Add custom header for testing
            add_header X-Proxy-By "Nginx" always;
            add_header X-Upstream-Addr $upstream_addr always;
            add_header X-Response-Time $upstream_response_time always;
        }

        # Static content (for testing)
        location /static/ {
            alias /usr/share/nginx/html/;
        }
    }
}
```

### Step 4: Docker Compose for Testing

```yaml
# src/test/resources/docker-compose.yml

version: '3.8'

services:
  # Spring Boot Application
  app:
    image: nginx-test-app:latest
    container_name: app-container
    build:
      context: ../../../
      dockerfile: docker/Dockerfile
    ports:
      - "8080"
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - SERVER_PORT=8080
    networks:
      - test-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    labels:
      - "test.service=app"

  # Nginx Reverse Proxy
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx-container
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - test-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    labels:
      - "test.service=nginx"

networks:
  test-network:
    driver: bridge
```

### Step 5: Dockerfile for Application

```dockerfile
# docker/Dockerfile

FROM eclipse-temurin:17-jdk-alpine AS build

WORKDIR /app

# Copy maven wrapper and pom.xml
COPY .mvn/ .mvn
COPY mvnw pom.xml ./

# Download dependencies
RUN ./mvnw dependency:go-offline

# Copy source code
COPY src ./src

# Build application
RUN ./mvnw clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copy JAR from build stage
COPY --from=build /app/target/*.jar app.jar

# Install curl for healthcheck
RUN apk add --no-cache curl

# Expose port
EXPOSE 8080

# Run application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 6: Integration Test Implementation

```java
package com.deepak.app.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import io.restassured.response.Response;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.DockerComposeContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.io.File;
import java.time.Duration;
import java.util.List;
import java.util.Map;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Nginx Integration Test
 * 
 * This test class demonstrates comprehensive integration testing
 * with Nginx as a reverse proxy using Testcontainers.
 * 
 * Test Flow:
 * 1. @BeforeAll: Start Docker containers (Nginx
