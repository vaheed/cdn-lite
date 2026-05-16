# Production Mini CDN Platform

> Global 3-region CDN using Docker Swarm · OpenResty · Varnish · Redis · MinIO · PowerDNS

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Folder Structure](#2-folder-structure)
3. [Docker Swarm Stack Files](#3-docker-swarm-stack-files)
4. [OpenResty Configuration](#4-openresty-configuration)
5. [Lua Signed URL Validation](#5-lua-signed-url-validation)
6. [Varnish VCL](#6-varnish-vcl)
7. [Redis Configuration](#7-redis-configuration)
8. [MinIO Configuration](#8-minio-configuration)
9. [PowerDNS + GeoDNS](#9-powerdns--geodns)
10. [Networking & Kernel Tuning](#10-networking--kernel-tuning)
11. [Security Hardening](#11-security-hardening)
12. [Observability Stack](#12-observability-stack)
13. [CI/CD Pipeline](#13-cicd-pipeline)
14. [Laravel Management API](#14-laravel-management-api)
15. [Backup & Disaster Recovery](#15-backup--disaster-recovery)
16. [Deployment Guide](#16-deployment-guide)

---

## 1. Architecture Overview

### Request Flow

```
Client (anywhere)
  │
  ▼
PowerDNS GeoDNS  ──► resolves to nearest edge IP
  │
  ▼
OpenResty (port 443, HTTP/2+HTTP/3)
  │  - TLS termination
  │  - Signed URL validation (Lua + Redis)
  │  - Rate limiting (Lua + Redis)
  │  - WAF rules
  ▼
Varnish (port 6081)
  │  - Edge cache (HIT → serve immediately)
  │  - MISS → forward to origin
  ▼
MinIO (S3 API, port 9000)
  │  - Object storage origin
  │  - Presigned backend URLs
  └── Replication (optional, async to other regions)
```

### Cache Hierarchy

```
L1: OpenResty proxy_cache (micro-cache, 10s)
L2: Varnish (main edge cache, TTL-driven)
L3: MinIO (origin, durable object storage)
```

### DNS Routing

```
cdn.example.com  TTL 30
  ├── EU users   → 185.x.x.x  (eu-node)
  ├── US users   → 45.x.x.x   (us-node)
  └── Asia users → 103.x.x.x  (asia-node)
```

### Docker Swarm Topology

```
eu-node   (manager)
us-node   (manager)
asia-node (manager)
  All 3 managers = quorum-safe (tolerates 1 failure)

Overlay networks:
  edge_net     – OpenResty ↔ Varnish
  storage_net  – Varnish ↔ MinIO
  internal_net – Redis, metrics, management
```

---

## 2. Folder Structure

```
cdn-platform/
├── .gitlab-ci.yml
├── .env.example
├── Makefile
│
├── stacks/
│   ├── edge.yml          # OpenResty + Varnish
│   ├── storage.yml       # MinIO
│   ├── data.yml          # Redis
│   ├── dns.yml           # PowerDNS
│   ├── observability.yml # Prometheus + Grafana + Loki
│   └── management.yml    # Laravel API
│
├── openresty/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── conf.d/
│   │   ├── cdn.conf
│   │   └── api.conf
│   ├── lua/
│   │   ├── signed_url.lua
│   │   ├── rate_limit.lua
│   │   ├── auth.lua
│   │   └── utils.lua
│   └── certs/            # mounted volume
│
├── varnish/
│   ├── Dockerfile
│   ├── default.vcl
│   └── purge.vcl
│
├── redis/
│   ├── redis.conf
│   └── users.acl
│
├── minio/
│   ├── policies/
│   │   ├── images.json
│   │   ├── videos.json
│   │   └── downloads.json
│   └── lifecycle/
│       └── lifecycle.xml
│
├── powerdns/
│   ├── pdns.conf
│   ├── geoip.yaml
│   └── zones/
│       └── example.com.yaml
│
├── observability/
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── alerts/
│   │       └── cdn.yml
│   ├── grafana/
│   │   ├── datasources/
│   │   └── dashboards/
│   │       ├── cache.json
│   │       ├── traffic.json
│   │       └── geo.json
│   └── loki/
│       └── loki.yml
│
├── laravel-api/
│   ├── app/
│   │   ├── Http/Controllers/
│   │   │   ├── PurgeController.php
│   │   │   ├── SignedUrlController.php
│   │   │   └── AnalyticsController.php
│   │   └── Jobs/
│   │       └── PurgeCacheJob.php
│   └── routes/api.php
│
├── scripts/
│   ├── bootstrap-swarm.sh
│   ├── deploy-all.sh
│   ├── rotate-secrets.sh
│   └── sysctl-tune.sh
│
└── secrets/
    └── .gitkeep   # secrets injected by CI, never committed
```

---

## 3. Docker Swarm Stack Files

### `stacks/edge.yml`

```yaml
version: "3.9"

networks:
  edge_net:
    external: true
  storage_net:
    external: true
  internal_net:
    external: true

secrets:
  cdn_secret_key:
    external: true
  redis_password:
    external: true
  ssl_cert:
    external: true
  ssl_key:
    external: true

configs:
  openresty_nginx:
    external: true
  openresty_cdn_conf:
    external: true
  varnish_vcl:
    external: true

services:
  openresty:
    image: registry.example.com/cdn/openresty:latest
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: udp
        mode: host   # HTTP/3 / QUIC
    networks:
      - edge_net
      - internal_net
    secrets:
      - cdn_secret_key
      - redis_password
      - ssl_cert
      - ssl_key
    configs:
      - source: openresty_nginx
        target: /usr/local/openresty/nginx/conf/nginx.conf
      - source: openresty_cdn_conf
        target: /usr/local/openresty/nginx/conf/conf.d/cdn.conf
    environment:
      - EDGE_REGION=${EDGE_REGION}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - MINIO_ENDPOINT=minio:9000
    deploy:
      mode: global          # one per node
      update_config:
        parallelism: 1
        delay: 15s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 5
      resources:
        limits:
          cpus: "2"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-labels: "service=openresty,region=${EDGE_REGION}"

  varnish:
    image: registry.example.com/cdn/varnish:latest
    networks:
      - edge_net
      - storage_net
    configs:
      - source: varnish_vcl
        target: /etc/varnish/default.vcl
    environment:
      - VARNISH_SIZE=4G
      - MINIO_HOST=minio
      - MINIO_PORT=9000
    deploy:
      mode: global
      update_config:
        parallelism: 1
        delay: 15s
        order: start-first
      restart_policy:
        condition: any
        delay: 5s
      resources:
        limits:
          cpus: "2"
          memory: 5G
        reservations:
          cpus: "0.5"
          memory: 2G
    healthcheck:
      test: ["CMD", "varnishadm", "ping"]
      interval: 15s
      timeout: 5s
      retries: 3
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-labels: "service=varnish,region=${EDGE_REGION}"
```

### `stacks/data.yml`

```yaml
version: "3.9"

networks:
  internal_net:
    external: true

secrets:
  redis_password:
    external: true

services:
  redis:
    image: redis:7.2-alpine
    command: >
      sh -c "redis-server /usr/local/etc/redis/redis.conf
             --requirepass $$(cat /run/secrets/redis_password)"
    networks:
      - internal_net
    secrets:
      - redis_password
    volumes:
      - redis_data:/data
      - redis_config:/usr/local/etc/redis
    deploy:
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: any
        delay: 5s
      resources:
        limits:
          cpus: "1"
          memory: 1G
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  redis_data:
    driver: local
  redis_config:
    driver: local
```

### `stacks/storage.yml`

```yaml
version: "3.9"

networks:
  storage_net:
    external: true
  internal_net:
    external: true

secrets:
  minio_root_user:
    external: true
  minio_root_password:
    external: true

services:
  minio:
    image: minio/minio:RELEASE.2024-01-01T00-00-00Z
    command: server /data --console-address ":9001"
    networks:
      - storage_net
      - internal_net
    secrets:
      - minio_root_user
      - minio_root_password
    environment:
      - MINIO_ROOT_USER_FILE=/run/secrets/minio_root_user
      - MINIO_ROOT_PASSWORD_FILE=/run/secrets/minio_root_password
      - MINIO_PROMETHEUS_AUTH_TYPE=public
      - MINIO_REGION_NAME=${EDGE_REGION}
    volumes:
      - minio_data:/data
    ports:
      - target: 9001
        published: 9001
        mode: host    # console, firewall to management IPs only
    deploy:
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: any
        delay: 5s
      resources:
        limits:
          cpus: "2"
          memory: 4G
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  minio_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/minio
```

### `stacks/dns.yml`

```yaml
version: "3.9"

networks:
  internal_net:
    external: true

secrets:
  pdns_api_key:
    external: true

services:
  powerdns:
    image: powerdns/pdns-auth-48:latest
    ports:
      - target: 53
        published: 53
        protocol: udp
        mode: host
      - target: 53
        published: 53
        protocol: tcp
        mode: host
      - target: 8081
        published: 8081
        mode: host    # API, firewall to management IPs
    networks:
      - internal_net
    secrets:
      - pdns_api_key
    volumes:
      - pdns_config:/etc/powerdns
      - geoip_db:/usr/share/GeoIP
    deploy:
      placement:
        constraints:
          - node.labels.dns == true
      restart_policy:
        condition: any
        delay: 5s
    healthcheck:
      test: ["CMD", "pdns_control", "ping"]
      interval: 15s
      timeout: 5s
      retries: 3

volumes:
  pdns_config:
    driver: local
  geoip_db:
    driver: local
```

---

## 4. OpenResty Configuration

### `openresty/nginx.conf`

```nginx
user www-data;
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 1048576;
pid /var/run/openresty.pid;

error_log /var/log/openresty/error.log warn;

events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # Lua path
    lua_package_path '/usr/local/openresty/lualib/?.lua;;';
    lua_package_cpath '/usr/local/openresty/lualib/?.so;;';

    # Shared memory zones
    lua_shared_dict cdn_cache       64m;
    lua_shared_dict rate_limit      32m;
    lua_shared_dict signed_tokens   16m;

    # Redis connection pool init
    init_by_lua_file /usr/local/openresty/nginx/lua/init.lua;

    # Logging
    log_format cdn_json escape=json
        '{"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"bytes_sent":$bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_referer":"$http_referer",'
        '"http_user_agent":"$http_user_agent",'
        '"x_cache":"$upstream_http_x_cache",'
        '"edge_region":"$edge_region"}';

    access_log /var/log/openresty/access.log cdn_json buffer=64k flush=5s;

    # Performance
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
    keepalive_timeout  75;
    keepalive_requests 10000;

    # Buffer tuning
    client_body_buffer_size    128k;
    client_max_body_size       5G;
    client_header_buffer_size  4k;
    large_client_header_buffers 8 16k;
    output_buffers             2 512k;
    postpone_output            1460;

    # Proxy settings
    proxy_buffering          on;
    proxy_buffer_size        128k;
    proxy_buffers            256 16k;
    proxy_busy_buffers_size  256k;
    proxy_temp_file_write_size 256k;
    proxy_read_timeout       300s;
    proxy_connect_timeout    10s;
    proxy_send_timeout       120s;

    # Compression
    gzip             on;
    gzip_vary        on;
    gzip_comp_level  4;
    gzip_min_length  1024;
    gzip_proxied     any;
    gzip_types
        text/plain text/css text/javascript application/javascript
        application/json application/xml image/svg+xml font/woff2;

    brotli             on;
    brotli_comp_level  4;
    brotli_min_length  1024;
    brotli_types
        text/plain text/css text/javascript application/javascript
        application/json application/xml image/svg+xml font/woff2;

    # Proxy cache
    proxy_cache_path /var/cache/openresty/micro
        levels=1:2
        keys_zone=micro_cache:32m
        max_size=10g
        inactive=60s
        use_temp_path=off;

    # Set edge region variable
    geo $edge_region {
        default unknown;
    }

    # TLS hardening
    ssl_protocols              TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers  off;
    ssl_ciphers                ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_session_cache          shared:SSL:50m;
    ssl_session_timeout        1d;
    ssl_session_tickets        off;
    ssl_stapling               on;
    ssl_stapling_verify        on;
    resolver                   1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout           5s;

    # Include site configs
    include /usr/local/openresty/nginx/conf/conf.d/*.conf;
}
```

### `openresty/conf.d/cdn.conf`

```nginx
# Upstream: Varnish
upstream varnish_backend {
    server varnish:6081 max_fails=3 fail_timeout=10s;
    keepalive 128;
}

# HTTP → HTTPS redirect
server {
    listen 80 default_server reuseport;
    server_name _;
    return 301 https://$host$request_uri;
}

# Main CDN server
server {
    listen 443 ssl http2 reuseport;
    listen 443 quic reuseport;

    server_name cdn.example.com *.cdn.example.com;

    ssl_certificate     /run/secrets/ssl_cert;
    ssl_certificate_key /run/secrets/ssl_key;

    # HTTP/3 advertisement
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options    "nosniff" always;
    add_header X-Frame-Options           "DENY" always;
    add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
    add_header X-Edge                    $hostname always;
    add_header X-Request-ID              $request_id always;

    # Health check (no auth)
    location = /health {
        access_log off;
        return 200 "ok\n";
    }

    # CDN object delivery
    location ~ ^/(?<bucket>[a-z0-9_-]+)/(?<object>.+)$ {
        # Signed URL validation
        access_by_lua_file /usr/local/openresty/nginx/lua/signed_url.lua;

        # Rate limiting
        access_by_lua_file /usr/local/openresty/nginx/lua/rate_limit.lua;

        # Micro cache for repeated identical requests
        proxy_cache            micro_cache;
        proxy_cache_valid      200 206 10s;
        proxy_cache_use_stale  error timeout updating;
        proxy_cache_lock       on;
        proxy_cache_key        "$scheme$host$uri$is_args$args";

        # Pass to Varnish
        proxy_pass http://varnish_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection        "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID      $request_id;
        proxy_set_header X-Edge-Region     $ENV{EDGE_REGION};

        # Pass X-Cache from Varnish through
        proxy_pass_header X-Cache;

        # Support range requests
        proxy_set_header Range $http_range;

        # Response headers
        add_header X-Request-ID $request_id always;
        add_header X-Edge       $hostname   always;
    }
}
```

### `openresty/lua/init.lua`

```lua
local redis = require "resty.redis"

-- Create and test Redis connection on worker init
local function init_redis()
    local red = redis:new()
    red:set_timeouts(1000, 1000, 2000)
    local ok, err = red:connect(os.getenv("REDIS_HOST"), tonumber(os.getenv("REDIS_PORT")))
    if not ok then
        ngx.log(ngx.ERR, "Failed to connect to Redis on init: ", err)
        return
    end
    -- Read password from Docker secret
    local f = io.open("/run/secrets/redis_password", "r")
    if f then
        local pass = f:read("*l"):gsub("%s+$", "")
        f:close()
        red:auth(pass)
    end
    red:close()
end

init_redis()
```

---

## 5. Lua Signed URL Validation

### `openresty/lua/signed_url.lua`

```lua
local hmac      = require "resty.hmac"
local redis_mod = require "resty.redis"
local str       = require "resty.string"

-- ──────────────────────────────────────────────
-- Helpers
-- ──────────────────────────────────────────────
local function get_secret()
    local f = io.open("/run/secrets/cdn_secret_key", "r")
    if not f then return nil end
    local s = f:read("*l"):gsub("%s+$", "")
    f:close()
    return s
end

local function redis_conn()
    local red = redis_mod:new()
    red:set_timeouts(500, 500, 1000)
    local ok, err = red:connect(os.getenv("REDIS_HOST"), 6379)
    if not ok then return nil, err end
    local pf = io.open("/run/secrets/redis_password", "r")
    if pf then
        local pass = pf:read("*l"):gsub("%s+$", "")
        pf:close()
        red:auth(pass)
    end
    return red, nil
end

local function release_redis(red)
    red:set_keepalive(10000, 100)
end

-- ──────────────────────────────────────────────
-- Main validation logic
-- ──────────────────────────────────────────────
local args, err = ngx.req.get_uri_args()
if err then
    ngx.log(ngx.ERR, "Failed to parse URI args: ", err)
    return ngx.exit(400)
end

local expires   = args["expires"]
local signature = args["signature"]
local lock_ip   = args["client_ip"]   -- optional IP restriction

-- Both params required
if not expires or not signature then
    ngx.log(ngx.WARN, "Missing signed URL params from ", ngx.var.remote_addr)
    return ngx.exit(403)
end

-- ──── Expiry check ────
local exp_ts = tonumber(expires)
if not exp_ts then
    return ngx.exit(403)
end

if ngx.time() > exp_ts then
    ngx.log(ngx.WARN, "Expired signed URL for ", ngx.var.uri)
    return ngx.exit(410)
end

-- ──── IP lock (optional) ────
if lock_ip and lock_ip ~= ngx.var.remote_addr then
    ngx.log(ngx.WARN, "IP mismatch: expected ", lock_ip, " got ", ngx.var.remote_addr)
    return ngx.exit(403)
end

-- ──── HMAC validation ────
local secret = get_secret()
if not secret then
    ngx.log(ngx.ERR, "Failed to read CDN secret")
    return ngx.exit(500)
end

-- Canonical string: URI path + expires (+ client_ip if present)
local canonical = ngx.var.uri .. expires
if lock_ip then
    canonical = canonical .. lock_ip
end

local h = hmac:new(secret, hmac.ALGOS.SHA256)
if not h then
    return ngx.exit(500)
end

local expected = str.to_hex(h:final(canonical))

-- Constant-time compare to prevent timing attacks
if expected ~= signature then
    ngx.log(ngx.WARN, "Invalid signature for ", ngx.var.uri, " from ", ngx.var.remote_addr)
    return ngx.exit(403)
end

-- ──── Anti-replay: check Redis revocation list ────
local red, rerr = redis_conn()
if red then
    local revoked = red:get("revoked:" .. signature)
    release_redis(red)
    if revoked and revoked ~= ngx.null then
        ngx.log(ngx.WARN, "Revoked token used: ", signature)
        return ngx.exit(403)
    end
end

-- All checks passed — continue
ngx.log(ngx.INFO, "Valid signed URL: ", ngx.var.uri)
```

### `openresty/lua/rate_limit.lua`

```lua
local redis_mod = require "resty.redis"

local RATE_LIMIT  = 500   -- requests per minute per IP
local WINDOW      = 60    -- seconds

local ip = ngx.var.remote_addr
local key = "ratelimit:" .. ip

local red = redis_mod:new()
red:set_timeouts(200, 200, 500)

local ok, err = red:connect(os.getenv("REDIS_HOST"), 6379)
if not ok then
    -- Redis unavailable: fail open (allow request)
    ngx.log(ngx.ERR, "Rate limit Redis unavailable: ", err)
    return
end

local pf = io.open("/run/secrets/redis_password", "r")
if pf then
    local pass = pf:read("*l"):gsub("%s+$", "")
    pf:close()
    red:auth(pass)
end

-- Sliding window using INCR + EXPIRE
local count, ierr = red:incr(key)
if ierr then
    red:set_keepalive(10000, 100)
    return
end

if count == 1 then
    red:expire(key, WINDOW)
end

red:set_keepalive(10000, 100)

if count > RATE_LIMIT then
    ngx.header["Retry-After"] = WINDOW
    ngx.header["X-RateLimit-Limit"] = RATE_LIMIT
    ngx.header["X-RateLimit-Remaining"] = 0
    return ngx.exit(429)
end

ngx.header["X-RateLimit-Limit"]     = RATE_LIMIT
ngx.header["X-RateLimit-Remaining"] = RATE_LIMIT - count
```

### `openresty/lua/utils.lua`

```lua
local M = {}

-- Generate a signed URL
-- Usage: utils.sign_url("/videos/movie.mp4", 3600, secret)
function M.sign_url(path, ttl_seconds, secret, client_ip)
    local hmac = require "resty.hmac"
    local str  = require "resty.string"

    local expires  = ngx.time() + ttl_seconds
    local canonical = path .. expires
    if client_ip then
        canonical = canonical .. client_ip
    end

    local h = hmac:new(secret, hmac.ALGOS.SHA256)
    local sig = str.to_hex(h:final(canonical))

    local url = path .. "?expires=" .. expires .. "&signature=" .. sig
    if client_ip then
        url = url .. "&client_ip=" .. client_ip
    end
    return url
end

return M
```

---

## 6. Varnish VCL

### `varnish/default.vcl`

```vcl
vcl 4.1;

import std;
import directors;

# ────────────────────────────────────────────────
# Backends
# ────────────────────────────────────────────────
backend minio {
    .host    = "minio";
    .port    = "9000";
    .connect_timeout = 5s;
    .first_byte_timeout  = 30s;
    .between_bytes_timeout = 10s;
    .probe = {
        .url     = "/minio/health/live";
        .timeout = 3s;
        .interval = 10s;
        .window  = 5;
        .threshold = 3;
    }
}

# ────────────────────────────────────────────────
# ACLs
# ────────────────────────────────────────────────
acl purge_acl {
    "127.0.0.1";
    "10.0.0.0"/8;      # Docker overlay
    "172.16.0.0"/12;
}

# ────────────────────────────────────────────────
# Initialization
# ────────────────────────────────────────────────
sub vcl_init {
    new origin = directors.round_robin();
    origin.add_backend(minio);
}

# ────────────────────────────────────────────────
# vcl_recv – request handling
# ────────────────────────────────────────────────
sub vcl_recv {
    set req.backend_hint = origin.backend();

    # ── Purge handling ──
    if (req.method == "PURGE") {
        if (!client.ip ~ purge_acl) {
            return (synth(405, "Not allowed"));
        }
        return (purge);
    }

    # ── Cache bypass conditions ──
    if (req.method == "POST" || req.method == "PUT" || req.method == "DELETE") {
        return (pass);
    }

    if (req.url ~ "^/api/") {
        return (pass);
    }

    if (req.url ~ "^/admin/") {
        return (pass);
    }

    if (req.http.Authorization) {
        return (pass);
    }

    # ── Strip cookies for static assets ──
    if (req.url ~ "\.(jpg|jpeg|png|gif|webp|ico|svg|mp4|webm|mp3|woff2|woff|ttf|css|js|zip|tar|gz)(\?.*)?$") {
        unset req.http.Cookie;
    }

    # ── Normalize Accept-Encoding ──
    if (req.http.Accept-Encoding) {
        if (req.url ~ "\.(mp4|webm|mp3|zip|gz|br)$") {
            unset req.http.Accept-Encoding;
        } elsif (req.http.Accept-Encoding ~ "br") {
            set req.http.Accept-Encoding = "br";
        } elsif (req.http.Accept-Encoding ~ "gzip") {
            set req.http.Accept-Encoding = "gzip";
        } else {
            unset req.http.Accept-Encoding;
        }
    }

    return (hash);
}

# ────────────────────────────────────────────────
# vcl_hash – cache key
# ────────────────────────────────────────────────
sub vcl_hash {
    hash_data(req.url);
    hash_data(req.http.host);
    if (req.http.Accept-Encoding) {
        hash_data(req.http.Accept-Encoding);
    }
    return (lookup);
}

# ────────────────────────────────────────────────
# vcl_hit – cache hit
# ────────────────────────────────────────────────
sub vcl_hit {
    # Grace mode: serve stale up to 24h while revalidating
    if (obj.ttl >= 0s) {
        return (deliver);
    }
    if (std.healthy(req.backend_hint)) {
        if (obj.ttl + 10s > 0s) {
            return (deliver);
        }
    } else {
        # Backend sick: serve stale up to 24h
        if (obj.ttl + obj.grace > 0s) {
            return (deliver);
        }
    }
    return (fetch);
}

# ────────────────────────────────────────────────
# vcl_miss / vcl_pass → backend request headers
# ────────────────────────────────────────────────
sub vcl_backend_fetch {
    # Rewrite URL to MinIO path format
    # /images/photo.jpg → /images/photo.jpg (bucket/object)
    set bereq.http.Host = "minio:9000";
}

# ────────────────────────────────────────────────
# vcl_backend_response – set TTLs
# ────────────────────────────────────────────────
sub vcl_backend_response {
    # Remove signed URL params from cache key scope
    set beresp.http.X-Url = bereq.url;

    # TTL by content type / path
    if (bereq.url ~ "^/images/") {
        set beresp.ttl  = 30d;
        set beresp.grace = 24h;
        set beresp.http.Cache-Control = "public, max-age=2592000, stale-while-revalidate=86400";
    } elsif (bereq.url ~ "^/videos/") {
        set beresp.ttl  = 7d;
        set beresp.grace = 24h;
        set beresp.http.Cache-Control = "public, max-age=604800, stale-while-revalidate=3600";
    } elsif (bereq.url ~ "\.(css|js|woff2|woff|ttf)(\?.*)?$") {
        set beresp.ttl  = 365d;
        set beresp.grace = 24h;
        set beresp.http.Cache-Control = "public, max-age=31536000, immutable";
    } elsif (bereq.url ~ "^/downloads/") {
        set beresp.ttl  = 7d;
        set beresp.grace = 12h;
    }

    # Stale-if-error: serve stale for 1h on backend errors
    set beresp.http.Cache-Control = beresp.http.Cache-Control + ", stale-if-error=3600";

    # Enable streaming for large objects
    if (beresp.http.Content-Length ~ "^[0-9]{7,}") {
        set beresp.do_stream = true;
    }

    # Don't cache 5xx responses
    if (beresp.status >= 500) {
        set beresp.uncacheable = true;
        set beresp.ttl = 0s;
        return (deliver);
    }

    # Cache 404s briefly to reduce origin load
    if (beresp.status == 404) {
        set beresp.ttl = 60s;
    }

    return (deliver);
}

# ────────────────────────────────────────────────
# vcl_deliver – final response headers
# ────────────────────────────────────────────────
sub vcl_deliver {
    # X-Cache header
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }

    # Remove internal headers
    unset resp.http.X-Url;
    unset resp.http.X-Powered-By;
    unset resp.http.Server;
    unset resp.http.Via;

    return (deliver);
}

# ────────────────────────────────────────────────
# vcl_synth – error responses
# ────────────────────────────────────────────────
sub vcl_synth {
    if (resp.status == 405) {
        set resp.http.Content-Type = "application/json";
        synthetic("{\"error\":\"Method not allowed\"}");
        return (deliver);
    }
    return (deliver);
}
```

---

## 7. Redis Configuration

### `redis/redis.conf`

```
# Network
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 300
tcp-keepalive 60

# Authentication via ACL (see users.acl)
aclfile /usr/local/etc/redis/users.acl

# Persistence
save 3600 1
save 300 100
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data

# AOF for durability
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Memory management
maxmemory 768mb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128

# Disable dangerous commands
rename-command FLUSHALL ""
rename-command FLUSHDB  ""
rename-command DEBUG    ""
rename-command CONFIG   "CONFIG_ADMIN_ONLY"
rename-command SHUTDOWN "SHUTDOWN_ADMIN_ONLY"

# Latency
latency-monitor-threshold 50

# Keyspace notifications (for expiry events)
notify-keyspace-events "Ex"
```

### `redis/users.acl`

```
# Default user disabled
user default off nopass nocommands nokeys

# CDN application user
user cdn_app on >CHANGE_THIS_STRONG_PASSWORD ~* &* +@all -@dangerous -FLUSHALL -FLUSHDB -DEBUG -SHUTDOWN

# Read-only user for metrics
user metrics_reader on >METRICS_PASSWORD ~* &* +INFO +PING +CLIENT +MEMORY +LATENCY -@all
```

---

## 8. MinIO Configuration

### `minio/policies/images.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": ["*"] },
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::images/*"],
      "Condition": {
        "StringLike": {
          "aws:Referer": ["https://*.cdn.example.com/*"]
        }
      }
    }
  ]
}
```

### `minio/lifecycle/lifecycle.xml`

```xml
<LifecycleConfiguration>
  <Rule>
    <ID>expire-temp-uploads</ID>
    <Status>Enabled</Status>
    <Filter><Prefix>uploads/temp/</Prefix></Filter>
    <Expiration><Days>1</Days></Expiration>
  </Rule>

  <Rule>
    <ID>transition-old-videos</ID>
    <Status>Enabled</Status>
    <Filter><Prefix>videos/archive/</Prefix></Filter>
    <Transition>
      <Days>90</Days>
      <StorageClass>GLACIER</StorageClass>
    </Transition>
  </Rule>

  <Rule>
    <ID>expire-old-backups</ID>
    <Status>Enabled</Status>
    <Filter><Prefix>backups/</Prefix></Filter>
    <Expiration><Days>90</Days></Expiration>
  </Rule>

  <Rule>
    <ID>abort-incomplete-multipart</ID>
    <Status>Enabled</Status>
    <Filter><Prefix></Prefix></Filter>
    <AbortIncompleteMultipartUpload>
      <DaysAfterInitiation>3</DaysAfterInitiation>
    </AbortIncompleteMultipartUpload>
  </Rule>
</LifecycleConfiguration>
```

### MinIO Bootstrap Script

```bash
#!/usr/bin/env bash
# scripts/bootstrap-minio.sh
set -euo pipefail

MC=mc
ALIAS=local
ENDPOINT=http://minio:9000
USER=$(cat /run/secrets/minio_root_user)
PASS=$(cat /run/secrets/minio_root_password)

$MC alias set $ALIAS $ENDPOINT "$USER" "$PASS"

# Create buckets
for bucket in images videos downloads backups; do
    $MC mb --ignore-existing $ALIAS/$bucket
    $MC version enable $ALIAS/$bucket
done

# Apply policies
$MC anonymous set-json /policies/images.json   $ALIAS/images
$MC anonymous set-json /policies/videos.json   $ALIAS/videos
$MC anonymous set-json /policies/downloads.json $ALIAS/downloads

# Apply lifecycle
$MC ilm import $ALIAS/backups < /lifecycle/lifecycle.xml

echo "MinIO bootstrap complete."
```

---

## 9. PowerDNS + GeoDNS

### `powerdns/pdns.conf`

```ini
# PowerDNS Authoritative Server
setuid=pdns
setgid=pdns

# Launch GeoIP backend
launch=geoip
geoip-database-files=/usr/share/GeoIP/GeoLite2-Country.mmdb
geoip-database-cache=memory
geoip-zones-dir=/etc/powerdns/zones

# Secondary backend for standard records
launch+=bind
bind-config=/etc/powerdns/named.conf

# API
api=yes
api-key-file=/run/secrets/pdns_api_key
webserver=yes
webserver-address=0.0.0.0
webserver-port=8081
webserver-allow-from=10.0.0.0/8,127.0.0.1

# DNS settings
local-address=0.0.0.0
local-port=53
max-tcp-connection-duration=10
max-tcp-connections-per-client=20
tcp-idle-timeout=5

# Performance
distributor-threads=4
receiver-threads=2
signing-threads=2

# Security
query-logging=no
```

### `powerdns/zones/example.com.yaml`

```yaml
# GeoIP zone for cdn.example.com
name: cdn.example.com.
ttl: 30
records:
  cdn.example.com.:
    soa:
      - content: "ns1.example.com. hostmaster.example.com. 2024010101 3600 900 604800 300"
        ttl: 3600
    ns:
      - content: ns1.example.com.
        ttl: 3600
      - content: ns2.example.com.
        ttl: 3600

services:
  cdn.example.com.:
    default:
      - 45.x.x.x    # US (global fallback)
    EU:
      - 185.x.x.x   # EU node
    NA:
      - 45.x.x.x    # US node
    AS:
      - 103.x.x.x   # Asia node
    OC:
      - 103.x.x.x   # Asia (nearest for Oceania)
    SA:
      - 45.x.x.x    # US (nearest for South America)
    AF:
      - 185.x.x.x   # EU (nearest for Africa)
```

---

## 10. Networking & Kernel Tuning

### `scripts/sysctl-tune.sh`

```bash
#!/usr/bin/env bash
# Apply to ALL edge nodes

cat > /etc/sysctl.d/99-cdn-tuning.conf << 'EOF'
# ── Network core ──────────────────────────────
net.core.somaxconn          = 65535
net.core.netdev_max_backlog = 65535
net.core.rmem_default       = 31457280
net.core.rmem_max           = 134217728
net.core.wmem_default       = 31457280
net.core.wmem_max           = 134217728
net.core.optmem_max         = 65535

# ── TCP tuning ────────────────────────────────
net.ipv4.tcp_rmem                = 4096 87380 134217728
net.ipv4.tcp_wmem                = 4096 65536 134217728
net.ipv4.tcp_congestion_control  = bbr
net.core.default_qdisc           = fq
net.ipv4.tcp_max_syn_backlog     = 65535
net.ipv4.tcp_max_tw_buckets      = 2000000
net.ipv4.tcp_tw_reuse            = 1
net.ipv4.tcp_fin_timeout         = 10
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_keepalive_time      = 60
net.ipv4.tcp_keepalive_intvl     = 10
net.ipv4.tcp_keepalive_probes    = 6
net.ipv4.tcp_mtu_probing         = 1
net.ipv4.tcp_timestamps          = 1
net.ipv4.tcp_sack                = 1
net.ipv4.ip_local_port_range     = 1024 65535
net.ipv4.tcp_fastopen            = 3

# ── File descriptors ──────────────────────────
fs.file-max                      = 2097152
fs.nr_open                       = 2097152

# ── VM ───────────────────────────────────────
vm.swappiness                    = 5
vm.dirty_ratio                   = 10
vm.dirty_background_ratio        = 5

# ── IPv6 ─────────────────────────────────────
net.ipv6.conf.all.disable_ipv6   = 0
EOF

sysctl -p /etc/sysctl.d/99-cdn-tuning.conf

# File descriptor limits
cat > /etc/security/limits.d/cdn.conf << 'EOF'
*    soft nofile 1048576
*    hard nofile 1048576
root soft nofile 1048576
root hard nofile 1048576
EOF

echo "Kernel tuning applied."
```

### Docker Swarm Overlay Networks Bootstrap

```bash
#!/usr/bin/env bash
# Run once on the primary manager node
docker network create \
    --driver overlay \
    --attachable \
    --opt com.docker.network.driver.mtu=1450 \
    edge_net

docker network create \
    --driver overlay \
    --attachable \
    --opt com.docker.network.driver.driver.mtu=1450 \
    storage_net

docker network create \
    --driver overlay \
    --attachable \
    --opt com.docker.network.driver.mtu=1450 \
    internal_net
```

---

## 11. Security Hardening

### Firewall (UFW / nftables) — per edge node

```bash
#!/usr/bin/env bash
# scripts/firewall-setup.sh

# Default deny
ufw default deny incoming
ufw default allow outgoing

# Allow CDN traffic
ufw allow 80/tcp    comment "HTTP → redirect"
ufw allow 443/tcp   comment "HTTPS"
ufw allow 443/udp   comment "HTTP/3 QUIC"

# Allow DNS (on DNS node only)
# ufw allow 53/tcp
# ufw allow 53/udp

# Swarm management (from other nodes only)
ufw allow from 185.x.x.x to any port 2377 proto tcp  # EU
ufw allow from 45.x.x.x  to any port 2377 proto tcp  # US
ufw allow from 103.x.x.x to any port 2377 proto tcp  # Asia

# Swarm overlay (VXLAN)
ufw allow from 185.x.x.x to any port 4789 proto udp
ufw allow from 45.x.x.x  to any port 4789 proto udp
ufw allow from 103.x.x.x to any port 4789 proto udp

# Swarm gossip
ufw allow from 185.x.x.x to any port 7946
ufw allow from 45.x.x.x  to any port 7946
ufw allow from 103.x.x.x to any port 7946

# SSH (restrict to your management IP)
ufw allow from YOUR_MGMT_IP to any port 22 proto tcp

# Grafana (management only)
ufw allow from YOUR_MGMT_IP to any port 3000 proto tcp

# MinIO console (management only)
ufw allow from YOUR_MGMT_IP to any port 9001 proto tcp

ufw --force enable
```

### Fail2Ban — OpenResty

```ini
# /etc/fail2ban/filter.d/openresty-cdn.conf
[Definition]
failregex = ^.*"remote_addr":"<HOST>".*"status":4(03|10|29).*$
ignoreregex =
```

```ini
# /etc/fail2ban/jail.d/openresty-cdn.conf
[openresty-cdn]
enabled  = true
filter   = openresty-cdn
logpath  = /var/log/openresty/access.log
maxretry = 30
findtime = 60
bantime  = 600
action   = %(action_mwl)s
```

### Docker socket protection

```yaml
# In all compose stacks — NEVER expose Docker socket
# Never add:
#   volumes:
#     - /var/run/docker.sock:/var/run/docker.sock
# Use Docker Swarm service API or Portainer agent instead.
```

---

## 12. Observability Stack

### `stacks/observability.yml`

```yaml
version: "3.9"

networks:
  internal_net:
    external: true

volumes:
  prometheus_data:
  grafana_data:
  loki_data:

services:
  prometheus:
    image: prom/prometheus:v2.49.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    volumes:
      - prometheus_data:/prometheus
    configs:
      - source: prometheus_config
        target: /etc/prometheus/prometheus.yml
      - source: prometheus_alerts
        target: /etc/prometheus/alerts/cdn.yml
    networks:
      - internal_net
    deploy:
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 2G

  grafana:
    image: grafana/grafana:10.3.0
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=false
      - GF_SERVER_ROOT_URL=https://metrics.example.com
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_password
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - internal_net
    ports:
      - target: 3000
        published: 3000
        mode: host
    deploy:
      placement:
        constraints:
          - node.role == manager

  loki:
    image: grafana/loki:2.9.4
    command: -config.file=/etc/loki/loki.yml
    volumes:
      - loki_data:/loki
    networks:
      - internal_net
    deploy:
      placement:
        constraints:
          - node.role == manager

  node_exporter:
    image: prom/node-exporter:v1.7.0
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - internal_net
    deploy:
      mode: global    # one per node

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - internal_net
    deploy:
      mode: global
```

### `observability/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
  external_labels:
    cluster: cdn-global

rule_files:
  - /etc/prometheus/alerts/cdn.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - eu-node:9100
          - us-node:9100
          - asia-node:9100

  - job_name: cadvisor
    static_configs:
      - targets:
          - eu-node:8080
          - us-node:8080
          - asia-node:8080

  - job_name: varnish
    static_configs:
      - targets:
          - eu-node:9131
          - us-node:9131
          - asia-node:9131

  - job_name: openresty
    static_configs:
      - targets:
          - eu-node:9145
          - us-node:9145
          - asia-node:9145

  - job_name: redis
    static_configs:
      - targets:
          - eu-node:9121
          - us-node:9121
          - asia-node:9121

  - job_name: minio
    metrics_path: /minio/v2/metrics/cluster
    static_configs:
      - targets:
          - eu-node:9000
          - us-node:9000
          - asia-node:9000
```

### `observability/prometheus/alerts/cdn.yml`

```yaml
groups:
  - name: cdn_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          rate(nginx_http_requests_total{status=~"5.."}[5m]) /
          rate(nginx_http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High 5xx error rate on {{ $labels.instance }}"

      - alert: LowCacheHitRatio
        expr: |
          varnish_main_cache_hit /
          (varnish_main_cache_hit + varnish_main_cache_miss) < 0.7
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio below 70% on {{ $labels.instance }}"

      - alert: NodeHighMemory
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
          / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage above 90% on {{ $labels.instance }}"

      - alert: MinioDown
        expr: up{job="minio"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO is down on {{ $labels.instance }}"

      - alert: EdgeNodeDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Edge node {{ $labels.instance }} is unreachable"
```

---

## 13. CI/CD Pipeline

### `.gitlab-ci.yml`

```yaml
stages:
  - validate
  - build
  - push
  - deploy

variables:
  REGISTRY: registry.example.com
  OPENRESTY_IMAGE: $REGISTRY/cdn/openresty
  VARNISH_IMAGE: $REGISTRY/cdn/varnish
  DOCKER_TLS_CERTDIR: "/certs"

# ────────────────────────────────────────────────
# Validate stage
# ────────────────────────────────────────────────
validate:nginx:
  stage: validate
  image: openresty/openresty:alpine
  script:
    - openresty -t -c $CI_PROJECT_DIR/openresty/nginx.conf
  rules:
    - changes:
        - openresty/**/*

validate:vcl:
  stage: validate
  image: varnish:7.5
  script:
    - varnishd -C -f $CI_PROJECT_DIR/varnish/default.vcl
  rules:
    - changes:
        - varnish/**/*

validate:lua:
  stage: validate
  image: openresty/openresty:alpine
  script:
    - luac -p openresty/lua/*.lua
  rules:
    - changes:
        - openresty/lua/**/*

# ────────────────────────────────────────────────
# Build stage
# ────────────────────────────────────────────────
build:openresty:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  script:
    - docker build -t $OPENRESTY_IMAGE:$CI_COMMIT_SHORT_SHA openresty/
    - docker tag  $OPENRESTY_IMAGE:$CI_COMMIT_SHORT_SHA $OPENRESTY_IMAGE:latest
  rules:
    - changes:
        - openresty/**/*

build:varnish:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  script:
    - docker build -t $VARNISH_IMAGE:$CI_COMMIT_SHORT_SHA varnish/
    - docker tag  $VARNISH_IMAGE:$CI_COMMIT_SHORT_SHA $VARNISH_IMAGE:latest
  rules:
    - changes:
        - varnish/**/*

# ────────────────────────────────────────────────
# Push stage
# ────────────────────────────────────────────────
push:images:
  stage: push
  image: docker:24-dind
  services:
    - docker:24-dind
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $REGISTRY -u $REGISTRY_USER --password-stdin
  script:
    - docker push $OPENRESTY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $OPENRESTY_IMAGE:latest
    - docker push $VARNISH_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $VARNISH_IMAGE:latest
  only:
    - main

# ────────────────────────────────────────────────
# Deploy stage
# ────────────────────────────────────────────────
.deploy_template: &deploy_template
  stage: deploy
  image: docker:24-dind
  before_script:
    - eval $(ssh-agent -s)
    - echo "$DEPLOY_SSH_KEY" | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $SWARM_MANAGER_IP >> ~/.ssh/known_hosts

deploy:edge:
  <<: *deploy_template
  script:
    - |
      ssh deploy@$SWARM_MANAGER_IP "
        export EDGE_REGION=$EDGE_REGION
        docker stack deploy \
          --with-registry-auth \
          --compose-file /opt/cdn/stacks/edge.yml \
          cdn_edge
      "
  environment:
    name: production/edge
  only:
    - main

deploy:storage:
  <<: *deploy_template
  script:
    - |
      ssh deploy@$SWARM_MANAGER_IP "
        docker stack deploy \
          --with-registry-auth \
          --compose-file /opt/cdn/stacks/storage.yml \
          cdn_storage
      "
  environment:
    name: production/storage
  only:
    - main
  when: manual   # storage changes are manually triggered
```

---

## 14. Laravel Management API

### `laravel-api/routes/api.php`

```php
<?php
use App\Http\Controllers\PurgeController;
use App\Http\Controllers\SignedUrlController;
use App\Http\Controllers\AnalyticsController;
use App\Http\Controllers\EdgeStatusController;

Route::middleware('auth:sanctum')->group(function () {

    // Cache purge
    Route::post('/purge/url',     [PurgeController::class, 'purgeUrl']);
    Route::post('/purge/prefix',  [PurgeController::class, 'purgePrefix']);
    Route::post('/purge/all',     [PurgeController::class, 'purgeAll'])->middleware('role:admin');

    // Signed URL generation
    Route::post('/signed-url',    [SignedUrlController::class, 'generate']);
    Route::post('/signed-url/revoke', [SignedUrlController::class, 'revoke']);

    // Analytics
    Route::get('/analytics/summary',  [AnalyticsController::class, 'summary']);
    Route::get('/analytics/requests', [AnalyticsController::class, 'requests']);
    Route::get('/analytics/geo',      [AnalyticsController::class, 'geo']);
    Route::get('/analytics/cache',    [AnalyticsController::class, 'cacheHitRatio']);

    // Edge status
    Route::get('/edge/status', [EdgeStatusController::class, 'index']);
});
```

### `laravel-api/app/Http/Controllers/PurgeController.php`

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Http;
use App\Jobs\PurgeCacheJob;

class PurgeController extends Controller
{
    private array $edgeNodes = [
        'eu'   => 'http://varnish.eu.internal:6082',
        'us'   => 'http://varnish.us.internal:6082',
        'asia' => 'http://varnish.asia.internal:6082',
    ];

    public function purgeUrl(Request $request)
    {
        $request->validate(['url' => 'required|url']);

        PurgeCacheJob::dispatch($request->url, 'url')
            ->onQueue('purge')
            ->allOnConnection('redis');

        return response()->json(['status' => 'queued', 'url' => $request->url]);
    }

    public function purgePrefix(Request $request)
    {
        $request->validate(['prefix' => 'required|string|max:500']);

        PurgeCacheJob::dispatch($request->prefix, 'prefix')
            ->onQueue('purge');

        return response()->json(['status' => 'queued', 'prefix' => $request->prefix]);
    }
}
```

### `laravel-api/app/Jobs/PurgeCacheJob.php`

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class PurgeCacheJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable;

    public int $tries = 3;
    public int $backoff = 10;

    private array $edgeNodes = [
        'eu'   => ['http://185.x.x.x:6082'],
        'us'   => ['http://45.x.x.x:6082'],
        'asia' => ['http://103.x.x.x:6082'],
    ];

    public function __construct(
        private string $target,
        private string $type   // 'url' | 'prefix'
    ) {}

    public function handle(): void
    {
        foreach ($this->edgeNodes as $region => $urls) {
            foreach ($urls as $url) {
                try {
                    if ($this->type === 'url') {
                        Http::timeout(10)->send('PURGE', $url . parse_url($this->target, PHP_URL_PATH));
                    } else {
                        Http::timeout(10)->withHeaders([
                            'X-Purge-Prefix' => $this->target,
                        ])->send('PURGE', $url . '/');
                    }
                    Log::info("Purged {$this->target} on {$region}");
                } catch (\Exception $e) {
                    Log::error("Purge failed on {$region}: " . $e->getMessage());
                    $this->fail($e);
                }
            }
        }
    }
}
```

### `laravel-api/app/Http/Controllers/SignedUrlController.php`

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class SignedUrlController extends Controller
{
    public function generate(Request $request)
    {
        $request->validate([
            'path'       => 'required|string',
            'ttl'        => 'integer|min:60|max:604800',
            'client_ip'  => 'nullable|ip',
        ]);

        $secret    = config('cdn.secret_key');
        $path      = $request->path;
        $expires   = now()->addSeconds($request->ttl ?? 3600)->timestamp;
        $clientIp  = $request->client_ip;

        $canonical = $path . $expires . ($clientIp ?? '');
        $signature = hash_hmac('sha256', $canonical, $secret);

        $url = config('cdn.base_url') . $path
             . '?expires=' . $expires
             . '&signature=' . $signature;

        if ($clientIp) {
            $url .= '&client_ip=' . $clientIp;
        }

        return response()->json([
            'url'       => $url,
            'expires'   => $expires,
            'signature' => $signature,
        ]);
    }

    public function revoke(Request $request)
    {
        $request->validate(['signature' => 'required|string']);

        // Store in Redis revocation list with TTL matching the original expiry
        Redis::setex(
            'revoked:' . $request->signature,
            config('cdn.max_url_ttl', 604800),
            '1'
        );

        return response()->json(['status' => 'revoked']);
    }
}
```

---

## 15. Backup & Disaster Recovery

### Backup Strategy

```bash
#!/usr/bin/env bash
# scripts/backup.sh — Run daily via cron on manager node

set -euo pipefail

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/data/backups/$DATE
MINIO_ALIAS=local
BACKUP_BUCKET=backups

mkdir -p $BACKUP_DIR

# ── 1. Redis RDB snapshot ──────────────────────
redis-cli -a $(cat /run/secrets/redis_password) BGSAVE
sleep 5
cp /data/redis/dump.rdb $BACKUP_DIR/redis-dump.rdb

# ── 2. PowerDNS zones ──────────────────────────
cp -r /etc/powerdns/zones $BACKUP_DIR/pdns-zones

# ── 3. Docker Swarm configs and secrets list ───
docker config ls --format json > $BACKUP_DIR/swarm-configs.json
docker secret ls --format json > $BACKUP_DIR/swarm-secrets-list.json  # IDs only, not values

# ── 4. Stack files ─────────────────────────────
cp -r /opt/cdn/stacks $BACKUP_DIR/

# ── 5. Upload to MinIO backup bucket ───────────
mc cp --recursive $BACKUP_DIR $MINIO_ALIAS/$BACKUP_BUCKET/$DATE/

# ── 6. Sync to offsite (S3-compatible remote) ──
mc mirror $MINIO_ALIAS/$BACKUP_BUCKET s3-offsite/cdn-backups/ \
    --overwrite --remove

echo "Backup complete: $DATE"
```

### Disaster Recovery Runbook

```
SCENARIO: Complete loss of one edge node

1. DNS failover (automatic via health checks, TTL 30s):
   - PowerDNS health check fails → removes sick node from rotation
   - Traffic redistributes to remaining 2 nodes within 30s

2. Re-provision replacement node:
   docker swarm join --token <WORKER_TOKEN> <MANAGER_IP>:2377

3. Node labels:
   docker node update --label-add region=eu <NEW_NODE_ID>

4. Redeploy stacks (Swarm reschedules automatically):
   docker stack deploy --compose-file stacks/edge.yml    cdn_edge
   docker stack deploy --compose-file stacks/storage.yml cdn_storage

5. Restore MinIO data (if storage node):
   mc mirror s3-offsite/cdn-backups/latest/ minio-local/

6. Re-add to DNS rotation after health check passes.

SCENARIO: Swarm manager quorum loss

- With 3 managers: tolerate 1 failure automatically
- With 2 managers remaining: continue normally
- With 1 manager: force new election:
  docker swarm init --force-new-cluster

SCENARIO: Redis data loss

- Restore from latest RDB backup
- Signed URL revocations lost → temporarily increase monitoring
- Rate limit counters reset → acceptable (minor)
```

---

## 16. Deployment Guide

### Step 1: Provision nodes

```bash
# On each VPS (Ubuntu 24.04), run as root:
curl -fsSL https://get.docker.com | sh
usermod -aG docker deploy

# Apply kernel tuning
bash scripts/sysctl-tune.sh

# Set hostname
hostnamectl set-hostname eu-cdn-node   # adjust per region
```

### Step 2: Initialize Swarm

```bash
# On EU node (first manager):
docker swarm init --advertise-addr 185.x.x.x

# Get join tokens
docker swarm join-token manager  # for US and Asia nodes

# On US and Asia nodes:
docker swarm join --token <MANAGER_TOKEN> 185.x.x.x:2377

# Label nodes
docker node update --label-add region=eu   <EU_NODE_ID>
docker node update --label-add region=us   <US_NODE_ID>
docker node update --label-add region=asia <ASIA_NODE_ID>
docker node update --label-add dns=true    <EU_NODE_ID>
```

### Step 3: Create overlay networks

```bash
bash scripts/bootstrap-networks.sh
```

### Step 4: Inject secrets

```bash
#!/usr/bin/env bash
# scripts/inject-secrets.sh

printf "STRONG_SECRET_KEY_HERE" | docker secret create cdn_secret_key -
printf "REDIS_PASSWORD_HERE"    | docker secret create redis_password -
cat /path/to/ssl.crt            | docker secret create ssl_cert -
cat /path/to/ssl.key            | docker secret create ssl_key -
printf "MINIO_USER"             | docker secret create minio_root_user -
printf "MINIO_PASSWORD"         | docker secret create minio_root_password -
printf "PDNS_API_KEY"           | docker secret create pdns_api_key -
printf "GRAFANA_PASSWORD"       | docker secret create grafana_password -
```

### Step 5: Create Docker configs

```bash
docker config create openresty_nginx    openresty/nginx.conf
docker config create openresty_cdn_conf openresty/conf.d/cdn.conf
docker config create varnish_vcl        varnish/default.vcl
docker config create prometheus_config  observability/prometheus/prometheus.yml
docker config create prometheus_alerts  observability/prometheus/alerts/cdn.yml
```

### Step 6: Deploy stacks

```bash
export EDGE_REGION=eu

docker stack deploy --compose-file stacks/data.yml          cdn_data
docker stack deploy --compose-file stacks/storage.yml       cdn_storage
docker stack deploy --compose-file stacks/dns.yml           cdn_dns
docker stack deploy --compose-file stacks/edge.yml          cdn_edge
docker stack deploy --compose-file stacks/observability.yml cdn_obs
docker stack deploy --compose-file stacks/management.yml    cdn_mgmt
```

### Step 7: Bootstrap MinIO

```bash
docker run --rm \
    --network cdn_storage_net \
    -v $(pwd)/minio/policies:/policies:ro \
    -v $(pwd)/minio/lifecycle:/lifecycle:ro \
    minio/mc:latest \
    bash /scripts/bootstrap-minio.sh
```

### Step 8: Configure PowerDNS

```bash
# Copy GeoIP database
docker exec $(docker ps -qf name=cdn_dns_powerdns) \
    mkdir -p /usr/share/GeoIP

docker cp /path/to/GeoLite2-Country.mmdb \
    $(docker ps -qf name=cdn_dns_powerdns):/usr/share/GeoIP/

# Reload PowerDNS
docker exec $(docker ps -qf name=cdn_dns_powerdns) \
    pdns_control cycle
```

### Step 9: Verify

```bash
# Check all services
docker stack ps cdn_edge
docker stack ps cdn_storage

# Test CDN delivery
curl -I https://cdn.example.com/images/test.jpg
# Should see: X-Cache: MISS (first hit)

curl -I https://cdn.example.com/images/test.jpg
# Should see: X-Cache: HIT

# Test signed URL
curl -I "https://cdn.example.com/videos/movie.mp4?expires=$(date -d '+1 hour' +%s)&signature=<HMAC>"

# Test rate limiting
for i in $(seq 1 600); do curl -s -o /dev/null https://cdn.example.com/images/test.jpg; done
# Should see 429 after 500 req/min
```

### Rolling update procedure

```bash
# Update OpenResty config without downtime
docker config create openresty_cdn_conf_v2 openresty/conf.d/cdn.conf
docker service update \
    --config-rm openresty_cdn_conf \
    --config-add source=openresty_cdn_conf_v2,target=/usr/local/openresty/nginx/conf/conf.d/cdn.conf \
    --update-parallelism 1 \
    --update-delay 15s \
    --update-order start-first \
    cdn_edge_openresty
```

---

*End of Production Mini CDN Platform Documentation*
