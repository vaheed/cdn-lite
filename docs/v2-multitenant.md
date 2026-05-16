# CDN Platform v2 — Multi-Tenant Public Platform

> Multi-site · Automatic SSL · Per-domain WAF · CLI management · POP expansion

---

## Table of Contents

1. [What's New in v2](#1-whats-new-in-v2)
2. [Architecture Changes](#2-architecture-changes)
3. [Domain & SSL Automation](#3-domain--ssl-automation)
4. [Multi-Tenant OpenResty Config](#4-multi-tenant-openresty-config)
5. [WAF — Per-Domain, Enable/Disable](#5-waf--per-domain-enabledisable)
6. [CLI Tool — `cdnctl`](#6-cli-tool--cdnctl)
7. [Redis Domain Registry Schema](#7-redis-domain-registry-schema)
8. [Laravel API — New Endpoints](#8-laravel-api--new-endpoints)
9. [Step-by-Step: Add a New Site](#9-step-by-step-add-a-new-site)
10. [Step-by-Step: Add a New POP](#10-step-by-step-add-a-new-pop)
11. [Step-by-Step: Remove a Site](#11-step-by-step-remove-a-site)
12. [Improvement Ideas](#12-improvement-ideas)
13. [Updated Folder Structure](#13-updated-folder-structure)

---

## 1. What's New in v2

| Feature | v1 | v2 |
|---|---|---|
| Multi-site support | ❌ Single domain | ✅ Unlimited domains |
| SSL management | Manual certs | ✅ Automatic Let's Encrypt via lua-resty-acme |
| WAF | ❌ | ✅ Per-domain on/off + rule sets |
| CLI tool | ❌ | ✅ `cdnctl` full management |
| Domain config store | Static files | ✅ Redis (live reload, no restart) |
| POP onboarding | Manual | ✅ Documented runbook |
| Config reload | Service restart | ✅ `nginx -s reload` hot reload |
| Wildcard SSL | Manual | ✅ DNS-01 challenge automation |

---

## 2. Architecture Changes

### Domain Config Flow

```
cdnctl add-domain cdn.customer.com --origin 1.2.3.4 --waf on
        │
        ▼
Laravel API → Redis  (domain registry)
        │
        ▼
OpenResty Lua  (reads Redis on every request, LRU-cached 30s)
        │
        ├── WAF check (per-domain flag)
        ├── Signed URL check (if enabled)
        ├── Rate limit
        └── proxy_pass → Varnish → MinIO / customer origin
```

### SSL Automation Flow

```
New domain added to registry
        │
        ▼
OpenResty detects no cert for domain
        │
        ▼
lua-resty-acme → Let's Encrypt ACME HTTP-01 challenge
        │
        ▼
Cert stored in Redis (shared across all workers)
        │
        ▼
Auto-renewed 30 days before expiry
```

### Multi-Origin Flow

```
Each domain has its own origin config in Redis:
  {
    "origin":      "customer-server.com:443",
    "origin_ssl":  true,
    "waf":         true,
    "cache_ttl":   86400,
    "signed_urls": false,
    "active":      true
  }
```

---

## 3. Domain & SSL Automation

### OpenResty Dockerfile

```dockerfile
# openresty/Dockerfile
FROM openresty/openresty:1.25.3-alpine

# Install OPM packages
RUN opm get fffonion/lua-resty-acme \
             openresty/lua-resty-redis \
             openresty/lua-resty-hmac \
             bungle/lua-resty-string \
             openresty/lua-resty-lrucache \
             openresty/lua-resty-lock

# Install luarocks packages
RUN apk add --no-cache luarocks5.1 && \
    luarocks-5.1 install lua-resty-http

COPY nginx.conf      /usr/local/openresty/nginx/conf/nginx.conf
COPY conf.d/         /usr/local/openresty/nginx/conf/conf.d/
COPY lua/            /usr/local/openresty/nginx/lua/

EXPOSE 80 443
```

### `openresty/conf.d/acme.conf` — ACME / Let's Encrypt

```nginx
# Shared storage for ACME certs (Redis-backed via lua-resty-acme)
lua_shared_dict acme          16m;
lua_shared_dict domain_config 32m;   # LRU cache of domain configs
lua_shared_dict waf_cache      8m;

# ACME initialization
init_by_lua_block {
    require("resty.acme.autossl").init({
        tos_accepted         = true,
        account_email        = os.getenv("ACME_EMAIL"),
        account_key_path     = "/etc/openresty/acme/account.key",
        domain_key_paths     = {},   -- auto-generate per domain
        storage_adapter      = "redis",
        storage_config       = {
            host     = os.getenv("REDIS_HOST"),
            port     = 6379,
            password = io.open("/run/secrets/redis_password"):read("*l"),
            database = 1,    -- DB 1 for ACME data
        },
        -- Production ACME URL
        api_uri = "https://acme-v02.api.letsencrypt.org/directory",
        -- Staging for testing:
        -- api_uri = "https://acme-staging-v02.api.letsencrypt.org/directory",
    })
}

init_worker_by_lua_block {
    require("resty.acme.autossl").init_worker()
}
```

### `openresty/conf.d/cdn.conf` — Dynamic Multi-Tenant Server

```nginx
# HTTP → HTTPS + ACME HTTP-01 challenge handler
server {
    listen 80 default_server reuseport;
    server_name _;

    # ACME HTTP-01 challenge (must be port 80, no redirect)
    location /.well-known/acme-challenge/ {
        content_by_lua_block {
            require("resty.acme.autossl").serve_http_challenge()
        }
    }

    # Redirect everything else
    location / {
        return 301 https://$host$request_uri;
    }
}

# Dynamic HTTPS — handles ALL domains in one server block
server {
    listen 443 ssl http2 default_server reuseport;
    listen 443 quic reuseport;
    server_name _;

    # Dynamic certificate selection via lua-resty-acme
    ssl_certificate_by_lua_block {
        require("resty.acme.autossl").ssl_certificate()
    }

    # Fallback self-signed for initial handshake before cert is issued
    ssl_certificate     /etc/openresty/fallback/fallback.crt;
    ssl_certificate_key /etc/openresty/fallback/fallback.key;

    ssl_protocols              TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers  off;
    ssl_session_cache          shared:SSL:50m;
    ssl_session_timeout        1d;
    ssl_session_tickets        off;
    ssl_stapling               on;
    ssl_stapling_verify        on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Request-ID $request_id always;

    # Resolve domain config from Redis, then route
    location / {
        access_by_lua_file  /usr/local/openresty/nginx/lua/domain_router.lua;
        proxy_pass          http://varnish_backend;
        proxy_http_version  1.1;
        proxy_set_header    Connection        "";
        proxy_set_header    Host              $host;
        proxy_set_header    X-Real-IP         $remote_addr;
        proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_set_header    X-Request-ID      $request_id;
        proxy_set_header    X-CDN-Domain      $host;
        proxy_pass_header   X-Cache;
        add_header          X-Edge $hostname always;
    }

    # Health endpoint (no auth, no WAF)
    location = /.cdn/health {
        access_log off;
        return 200 "ok\n";
    }
}
```

---

## 4. Multi-Tenant OpenResty Config

### `openresty/lua/domain_router.lua`

```lua
local lrucache   = require "resty.lrucache"
local redis_mod  = require "resty.redis"
local cjson      = require "cjson.safe"
local waf        = require "cdn.waf"
local signed_url = require "cdn.signed_url"
local rate_limit = require "cdn.rate_limit"

-- LRU cache: up to 1000 domain configs, 30s TTL
local domain_cache, err = lrucache.new(1000)
if not domain_cache then
    ngx.log(ngx.ERR, "LRU cache init failed: ", err)
    return ngx.exit(500)
end

-- ──────────────────────────────────────────────
-- Redis helpers
-- ──────────────────────────────────────────────
local function redis_get(key)
    local red = redis_mod:new()
    red:set_timeouts(200, 200, 500)
    local ok, e = red:connect(os.getenv("REDIS_HOST"), 6379)
    if not ok then return nil, e end
    local pf = io.open("/run/secrets/redis_password", "r")
    if pf then red:auth(pf:read("*l"):gsub("%s+$", "")); pf:close() end
    red:select(0)  -- DB 0 for domain config
    local val = red:get(key)
    red:set_keepalive(10000, 100)
    return val
end

-- ──────────────────────────────────────────────
-- Load domain config (LRU-cached 30s)
-- ──────────────────────────────────────────────
local function get_domain_config(host)
    -- Strip port if present
    local domain = host:match("^([^:]+)")

    -- Check LRU cache first
    local config = domain_cache:get(domain)
    if config then
        return config
    end

    -- Miss → fetch from Redis
    local raw, e = redis_get("domain:" .. domain)
    if not raw or raw == ngx.null then
        return nil, "domain not found: " .. domain
    end

    local cfg = cjson.decode(raw)
    if not cfg then
        return nil, "invalid domain config JSON"
    end

    -- Cache for 30 seconds
    domain_cache:set(domain, cfg, 30)
    return cfg
end

-- ──────────────────────────────────────────────
-- Main routing logic
-- ──────────────────────────────────────────────
local host = ngx.var.host
local cfg, err = get_domain_config(host)

if not cfg then
    ngx.log(ngx.WARN, "Unknown domain: ", host, " error: ", err)
    return ngx.exit(444)   -- Connection closed — stealth drop for unknown domains
end

if not cfg.active then
    return ngx.exit(503)
end

-- ── Rate limiting (always on) ──────────────────
rate_limit.check(ngx.var.remote_addr, cfg.rate_limit or 500)

-- ── WAF check (per-domain) ────────────────────
if cfg.waf then
    waf.check()
end

-- ── Signed URL validation (per-domain) ────────
if cfg.signed_urls then
    signed_url.validate()
end

-- ── Set upstream origin for Varnish passthrough ──
-- Varnish reads X-CDN-Origin header to select backend
ngx.req.set_header("X-CDN-Origin",      cfg.origin)
ngx.req.set_header("X-CDN-Origin-SSL",  cfg.origin_ssl and "1" or "0")
ngx.req.set_header("X-CDN-Cache-TTL",   tostring(cfg.cache_ttl or 86400))
```

---

## 5. WAF — Per-Domain, Enable/Disable

### `openresty/lua/cdn/waf.lua`

```lua
-- cdn/waf.lua
-- Lightweight WAF: SQLi, XSS, path traversal, bad bots, method enforcement
-- Per-domain enable/disable via Redis domain config

local M = {}

-- ── Rule sets ──────────────────────────────────
local RULES = {
    -- SQL Injection
    {
        id   = "WAF001",
        name = "SQLi basic",
        pattern = "(%27)|(%22)|(--)|(%3B)|(union%20)|(select%20)|(insert%20)|(drop%20)|(update%20)|(delete%20)|(exec%20)|(cast%20)",
        target  = "uri_args",
        action  = "block",
    },
    -- XSS
    {
        id   = "WAF002",
        name = "XSS basic",
        pattern = "(<script)|(</script>)|(javascript:)|(onerror=)|(onload=)|(eval%28)|(alert%28)",
        target  = "uri_args",
        action  = "block",
    },
    -- Path traversal
    {
        id   = "WAF003",
        name = "Path traversal",
        pattern = "(\\.\\./)|(%2e%2e%2f)|(%2e%2e/)|(\\.%2e/)",
        target  = "uri",
        action  = "block",
    },
    -- Bad bots
    {
        id   = "WAF004",
        name = "Bad bots",
        pattern = "(nikto)|(sqlmap)|(nmap)|(masscan)|(zgrab)|(dirbuster)|(gobuster)|(nuclei)",
        target  = "user_agent",
        action  = "block",
    },
    -- HTTP method whitelist
    {
        id      = "WAF005",
        name    = "Method whitelist",
        allowed = { GET=true, HEAD=true, OPTIONS=true, POST=true, PUT=true, DELETE=true, PATCH=true },
        target  = "method",
        action  = "block",
    },
    -- Large request body (potential DoS)
    {
        id      = "WAF006",
        name    = "Body size limit",
        max     = 10 * 1024 * 1024,  -- 10 MB
        target  = "body_size",
        action  = "block",
    },
    -- Scanner signatures in URI
    {
        id   = "WAF007",
        name = "Scanner paths",
        pattern = "(/wp-admin)|(/phpmyadmin)|(/adminer)|(\\.env)|(\\.git/)|(/etc/passwd)|(/.ssh/)",
        target  = "uri",
        action  = "block",
    },
}

local function log_block(rule_id, rule_name, value)
    ngx.log(ngx.WARN, string.format(
        "[WAF BLOCK] rule=%s name=%s ip=%s host=%s uri=%s matched=%s",
        rule_id, rule_name,
        ngx.var.remote_addr,
        ngx.var.host,
        ngx.var.uri,
        tostring(value):sub(1, 100)
    ))
end

function M.check()
    local uri        = ngx.var.uri:lower()
    local args       = ngx.var.query_string or ""
    local ua         = (ngx.var.http_user_agent or ""):lower()
    local method     = ngx.var.request_method
    local body_size  = tonumber(ngx.var.content_length) or 0

    for _, rule in ipairs(RULES) do
        local matched = false
        local matched_val = ""

        if rule.target == "uri" then
            if uri:match(rule.pattern) then
                matched = true; matched_val = uri
            end

        elseif rule.target == "uri_args" then
            local decoded = ngx.unescape_uri(args):lower()
            if decoded:match(rule.pattern) then
                matched = true; matched_val = decoded:sub(1, 200)
            end

        elseif rule.target == "user_agent" then
            if ua:match(rule.pattern) then
                matched = true; matched_val = ua
            end

        elseif rule.target == "method" then
            if not rule.allowed[method] then
                matched = true; matched_val = method
            end

        elseif rule.target == "body_size" then
            if body_size > rule.max then
                matched = true; matched_val = tostring(body_size)
            end
        end

        if matched then
            log_block(rule.id, rule.name, matched_val)
            ngx.header["X-WAF-Block"] = rule.id
            return ngx.exit(403)
        end
    end
end

-- Check if WAF is enabled for a domain (called from domain_router)
function M.is_enabled(domain)
    -- Already resolved by domain_router from Redis cfg.waf
    return true
end

return M
```

---

## 6. CLI Tool — `cdnctl`

### Installation

```bash
# Install on any manager node or your workstation
curl -fsSL https://cdn.example.com/cli/install.sh | bash
# Or copy manually:
cp scripts/cdnctl /usr/local/bin/cdnctl
chmod +x /usr/local/bin/cdnctl
```

### `scripts/cdnctl`

```bash
#!/usr/bin/env bash
# cdnctl — CDN Platform CLI
# Version 2.0

set -euo pipefail

API_BASE="${CDNCTL_API:-https://api.cdn.example.com}"
TOKEN_FILE="${HOME}/.cdnctl_token"
VERSION="2.0.0"

# ──────────────────────────────────────────────
# Colors
# ──────────────────────────────────────────────
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; NC='\033[0m'; BOLD='\033[1m'

info()    { echo -e "${CYAN}[INFO]${NC}  $*"; }
success() { echo -e "${GREEN}[OK]${NC}    $*"; }
warn()    { echo -e "${YELLOW}[WARN]${NC}  $*"; }
error()   { echo -e "${RED}[ERROR]${NC} $*" >&2; exit 1; }

# ──────────────────────────────────────────────
# Auth helpers
# ──────────────────────────────────────────────
get_token() {
    [[ -f "$TOKEN_FILE" ]] || error "Not logged in. Run: cdnctl login"
    cat "$TOKEN_FILE"
}

api() {
    local method="$1"; shift
    local path="$1";   shift
    local data="${1:-}"
    local token
    token=$(get_token)

    if [[ -n "$data" ]]; then
        curl -sf -X "$method" \
            -H "Authorization: Bearer $token" \
            -H "Content-Type: application/json" \
            -d "$data" \
            "${API_BASE}${path}"
    else
        curl -sf -X "$method" \
            -H "Authorization: Bearer $token" \
            "${API_BASE}${path}"
    fi
}

# ──────────────────────────────────────────────
# Commands
# ──────────────────────────────────────────────

cmd_help() {
    echo -e "${BOLD}cdnctl${NC} v${VERSION} — CDN Platform CLI"
    echo ""
    echo -e "${BOLD}USAGE:${NC}"
    echo "  cdnctl <command> [options]"
    echo ""
    echo -e "${BOLD}AUTH:${NC}"
    echo "  login                          Authenticate and store token"
    echo "  logout                         Remove stored token"
    echo "  whoami                         Show current user"
    echo ""
    echo -e "${BOLD}DOMAIN MANAGEMENT:${NC}"
    echo "  domain add <domain> [opts]     Add a domain to the CDN"
    echo "  domain remove <domain>         Remove a domain"
    echo "  domain list                    List all domains"
    echo "  domain show <domain>           Show domain config"
    echo "  domain enable  <domain>        Enable a domain"
    echo "  domain disable <domain>        Disable a domain (returns 503)"
    echo "  domain set-origin <domain> <origin>  Change origin server"
    echo ""
    echo -e "${BOLD}SSL:${NC}"
    echo "  ssl status <domain>            Show certificate status"
    echo "  ssl renew  <domain>            Force certificate renewal"
    echo "  ssl list                       List all certificates"
    echo ""
    echo -e "${BOLD}WAF:${NC}"
    echo "  waf enable  <domain>           Enable WAF for domain"
    echo "  waf disable <domain>           Disable WAF for domain"
    echo "  waf status  <domain>           Show WAF status"
    echo "  waf rules                      List all WAF rules"
    echo ""
    echo -e "${BOLD}CACHE:${NC}"
    echo "  cache purge-url  <url>         Purge a specific URL"
    echo "  cache purge-prefix <domain> <prefix>  Purge by prefix"
    echo "  cache purge-domain <domain>    Purge all cache for domain"
    echo "  cache stats <domain>           Show cache hit/miss stats"
    echo ""
    echo -e "${BOLD}EDGE / POPs:${NC}"
    echo "  pop list                       List all POP nodes"
    echo "  pop status                     Show health of all POPs"
    echo "  pop add <name> <ip> <region>   Register a new POP"
    echo "  pop drain <name>               Drain a POP (remove from DNS)"
    echo "  pop undrain <name>             Re-add POP to DNS rotation"
    echo ""
    echo -e "${BOLD}ANALYTICS:${NC}"
    echo "  stats summary                  Global traffic summary"
    echo "  stats domain <domain>          Per-domain statistics"
    echo "  stats top                      Top 10 objects by requests"
    echo "  stats geo                      Traffic by geography"
    echo ""
    echo -e "${BOLD}SIGNED URLS:${NC}"
    echo "  sign <path> [--ttl=3600] [--ip=x.x.x.x]  Generate signed URL"
    echo "  revoke <signature>             Revoke a signed URL"
    echo ""
    echo -e "${BOLD}OPTIONS for domain add:${NC}"
    echo "  --origin    Origin server (required)  e.g. 1.2.3.4 or origin.example.com"
    echo "  --origin-ssl                          Use HTTPS to origin"
    echo "  --waf                                 Enable WAF"
    echo "  --signed-urls                         Require signed URLs"
    echo "  --cache-ttl=N                         Cache TTL in seconds (default 86400)"
    echo "  --rate-limit=N                        Requests/min per IP (default 500)"
    echo ""
    echo -e "${BOLD}EXAMPLES:${NC}"
    echo "  cdnctl domain add customer.com --origin 10.0.0.5 --waf"
    echo "  cdnctl waf enable customer.com"
    echo "  cdnctl cache purge-url https://customer.com/images/hero.jpg"
    echo "  cdnctl ssl status customer.com"
    echo "  cdnctl sign /videos/movie.mp4 --ttl=7200"
    echo "  cdnctl pop add us-east 45.1.2.3 us"
}

cmd_login() {
    read -r -p "Email: " email
    read -r -s -p "Password: " password; echo ""
    local token
    token=$(curl -sf -X POST \
        -H "Content-Type: application/json" \
        -d "{\"email\":\"$email\",\"password\":\"$password\"}" \
        "${API_BASE}/auth/token" | jq -r .token)
    [[ -z "$token" || "$token" == "null" ]] && error "Login failed."
    echo "$token" > "$TOKEN_FILE"
    chmod 600 "$TOKEN_FILE"
    success "Logged in as $email"
}

cmd_domain_add() {
    local domain="$1"; shift
    local origin="" origin_ssl="false" waf="false"
    local signed_urls="false" cache_ttl=86400 rate_limit=500

    while [[ $# -gt 0 ]]; do
        case "$1" in
            --origin=*)      origin="${1#*=}" ;;
            --origin)        origin="$2"; shift ;;
            --origin-ssl)    origin_ssl="true" ;;
            --waf)           waf="true" ;;
            --signed-urls)   signed_urls="true" ;;
            --cache-ttl=*)   cache_ttl="${1#*=}" ;;
            --rate-limit=*)  rate_limit="${1#*=}" ;;
        esac
        shift
    done

    [[ -z "$origin" ]] && error "Missing --origin"
    [[ -z "$domain" ]] && error "Missing domain name"

    info "Adding domain ${BOLD}$domain${NC} → origin: $origin"

    local payload
    payload=$(cat <<EOF
{
  "domain":      "$domain",
  "origin":      "$origin",
  "origin_ssl":  $origin_ssl,
  "waf":         $waf,
  "signed_urls": $signed_urls,
  "cache_ttl":   $cache_ttl,
  "rate_limit":  $rate_limit,
  "active":      true
}
EOF
)
    api POST /api/domains "" <<< "$payload" | jq .
    echo ""
    success "Domain $domain added."
    info "Point your DNS A/CNAME to the CDN edge IPs."
    info "SSL certificate will be issued automatically on first HTTPS request."
}

cmd_domain_list() {
    info "All domains:"
    api GET /api/domains | jq -r \
        '.[] | [.domain, .origin, (if .active then "✓ active" else "✗ disabled" end), (if .waf then "WAF:on" else "WAF:off" end), .ssl_status] | @tsv' \
        | column -t -s $'\t'
}

cmd_domain_show() {
    local domain="$1"
    api GET "/api/domains/$domain" | jq .
}

cmd_waf_enable() {
    local domain="$1"
    api PATCH "/api/domains/$domain" '{"waf":true}' | jq .
    success "WAF enabled for $domain"
}

cmd_waf_disable() {
    local domain="$1"
    api PATCH "/api/domains/$domain" '{"waf":false}' | jq .
    warn "WAF disabled for $domain"
}

cmd_waf_status() {
    local domain="$1"
    local waf
    waf=$(api GET "/api/domains/$domain" | jq -r .waf)
    if [[ "$waf" == "true" ]]; then
        echo -e "WAF status for ${BOLD}$domain${NC}: ${GREEN}ENABLED${NC}"
    else
        echo -e "WAF status for ${BOLD}$domain${NC}: ${RED}DISABLED${NC}"
    fi
}

cmd_cache_purge_url() {
    local url="$1"
    info "Purging URL: $url"
    api POST /api/purge/url "{\"url\":\"$url\"}" | jq .
    success "Purge queued on all POPs."
}

cmd_cache_purge_prefix() {
    local domain="$1"; local prefix="$2"
    info "Purging prefix: $prefix on $domain"
    api POST /api/purge/prefix "{\"domain\":\"$domain\",\"prefix\":\"$prefix\"}" | jq .
    success "Prefix purge queued."
}

cmd_cache_purge_domain() {
    local domain="$1"
    warn "This will purge ALL cached objects for $domain."
    read -r -p "Confirm? [y/N] " confirm
    [[ "$confirm" != "y" ]] && { info "Aborted."; exit 0; }
    api POST /api/purge/domain "{\"domain\":\"$domain\"}" | jq .
    success "Full domain purge queued."
}

cmd_ssl_status() {
    local domain="$1"
    api GET "/api/ssl/$domain" | jq .
}

cmd_ssl_renew() {
    local domain="$1"
    api POST "/api/ssl/$domain/renew" | jq .
    success "Renewal triggered for $domain"
}

cmd_sign() {
    local path="$1"; shift
    local ttl=3600 ip=""
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --ttl=*)  ttl="${1#*=}" ;;
            --ip=*)   ip="${1#*=}" ;;
        esac
        shift
    done
    local payload="{\"path\":\"$path\",\"ttl\":$ttl"
    [[ -n "$ip" ]] && payload+=",\"client_ip\":\"$ip\""
    payload+="}"
    api POST /api/signed-url "$payload" | jq .
}

cmd_pop_list() {
    api GET /api/pops | jq -r \
        '.[] | [.name, .ip, .region, .status, (.load | tostring + "%")] | @tsv' \
        | column -t -s $'\t'
}

cmd_pop_add() {
    local name="$1" ip="$2" region="$3"
    api POST /api/pops "{\"name\":\"$name\",\"ip\":\"$ip\",\"region\":\"$region\"}" | jq .
    success "POP $name ($ip) registered in region $region"
    info "Next steps:"
    info "  1. Run the bootstrap script on the new node"
    info "  2. cdnctl pop undrain $name  (after bootstrap completes)"
}

cmd_pop_status() {
    api GET /api/pops/status | jq -r \
        '.[] | [.name, .region, .ip, .latency_ms, .status, .ssl_ok, .cache_hit_pct] | @tsv' \
        | column -t -s $'\t'
}

cmd_stats_summary() {
    api GET /api/analytics/summary | jq .
}

cmd_stats_domain() {
    local domain="$1"
    api GET "/api/analytics/domain/$domain" | jq .
}

# ──────────────────────────────────────────────
# Dispatcher
# ──────────────────────────────────────────────
main() {
    [[ $# -eq 0 ]] && { cmd_help; exit 0; }
    local cmd="$1"; shift || true

    case "$cmd" in
        help|-h|--help)  cmd_help ;;
        login)           cmd_login ;;
        logout)          rm -f "$TOKEN_FILE"; success "Logged out." ;;
        whoami)          api GET /api/auth/me | jq -r .email ;;

        domain)
            local sub="${1:-}"; shift || true
            case "$sub" in
                add)     cmd_domain_add "$@" ;;
                remove)  api DELETE "/api/domains/$1" | jq . ;;
                list)    cmd_domain_list ;;
                show)    cmd_domain_show "$1" ;;
                enable)  api PATCH "/api/domains/$1" '{"active":true}'  | jq . ;;
                disable) api PATCH "/api/domains/$1" '{"active":false}' | jq . ;;
                set-origin) api PATCH "/api/domains/$1" "{\"origin\":\"$2\"}" | jq . ;;
                *)       error "Unknown domain subcommand: $sub" ;;
            esac ;;

        waf)
            local sub="${1:-}"; shift || true
            case "$sub" in
                enable)  cmd_waf_enable  "$1" ;;
                disable) cmd_waf_disable "$1" ;;
                status)  cmd_waf_status  "$1" ;;
                rules)   api GET /api/waf/rules | jq . ;;
                *)       error "Unknown waf subcommand: $sub" ;;
            esac ;;

        cache)
            local sub="${1:-}"; shift || true
            case "$sub" in
                purge-url)    cmd_cache_purge_url    "$1" ;;
                purge-prefix) cmd_cache_purge_prefix "$1" "$2" ;;
                purge-domain) cmd_cache_purge_domain "$1" ;;
                stats)        api GET "/api/analytics/cache/$1" | jq . ;;
                *)            error "Unknown cache subcommand: $sub" ;;
            esac ;;

        ssl)
            local sub="${1:-}"; shift || true
            case "$sub" in
                status) cmd_ssl_status "$1" ;;
                renew)  cmd_ssl_renew  "$1" ;;
                list)   api GET /api/ssl | jq . ;;
                *)      error "Unknown ssl subcommand: $sub" ;;
            esac ;;

        sign)    cmd_sign "$@" ;;
        revoke)  api POST /api/signed-url/revoke "{\"signature\":\"$1\"}" | jq . ;;

        pop)
            local sub="${1:-}"; shift || true
            case "$sub" in
                list)    cmd_pop_list ;;
                status)  cmd_pop_status ;;
                add)     cmd_pop_add "$1" "$2" "$3" ;;
                drain)   api POST "/api/pops/$1/drain"   | jq . ;;
                undrain) api POST "/api/pops/$1/undrain" | jq . ;;
                *)       error "Unknown pop subcommand: $sub" ;;
            esac ;;

        stats)
            local sub="${1:-}"; shift || true
            case "$sub" in
                summary) cmd_stats_summary ;;
                domain)  cmd_stats_domain "$1" ;;
                top)     api GET /api/analytics/top | jq . ;;
                geo)     api GET /api/analytics/geo | jq . ;;
                *)       error "Unknown stats subcommand: $sub" ;;
            esac ;;

        version) echo "cdnctl v$VERSION" ;;
        *)       error "Unknown command: $cmd. Run 'cdnctl help'" ;;
    esac
}

main "$@"
```

---

## 7. Redis Domain Registry Schema

```
DB 0 — Domain configs
  KEY:  domain:<fqdn>
  TYPE: string (JSON)
  TTL:  none (permanent until deleted)
  VAL:  {
          "domain":      "customer.com",
          "origin":      "52.1.2.3",
          "origin_ssl":  false,
          "origin_port": 80,
          "waf":         true,
          "signed_urls": false,
          "cache_ttl":   86400,
          "rate_limit":  500,
          "active":      true,
          "created_at":  1735000000,
          "ssl_status":  "active",        -- "pending"|"active"|"error"
          "ssl_expiry":  1766536000
        }

  KEY:  domains:index     (sorted set, all known domains)
  KEY:  domain:stats:<fqdn>:hits    (counter, daily key)
  KEY:  domain:stats:<fqdn>:miss    (counter, daily key)
  KEY:  domain:stats:<fqdn>:bytes   (counter, daily key)

DB 1 — ACME / SSL data (managed by lua-resty-acme)
  KEY:  acme:cert:<fqdn>
  KEY:  acme:key:<fqdn>
  KEY:  acme:account

DB 2 — Rate limiting
  KEY:  ratelimit:<ip>      TTL: 60s

DB 3 — Signed URL revocations
  KEY:  revoked:<signature>  TTL: max_url_ttl
```

---

## 8. Laravel API — New Endpoints

### Domain management routes (additions)

```php
// routes/api.php additions
Route::middleware('auth:sanctum')->group(function () {

    // Domain CRUD
    Route::get   ('/domains',          [DomainController::class, 'index']);
    Route::post  ('/domains',          [DomainController::class, 'store']);
    Route::get   ('/domains/{domain}', [DomainController::class, 'show']);
    Route::patch ('/domains/{domain}', [DomainController::class, 'update']);
    Route::delete('/domains/{domain}', [DomainController::class, 'destroy']);

    // SSL
    Route::get ('/ssl',              [SslController::class, 'index']);
    Route::get ('/ssl/{domain}',     [SslController::class, 'status']);
    Route::post('/ssl/{domain}/renew',[SslController::class, 'renew']);

    // WAF rules
    Route::get('/waf/rules', [WafController::class, 'rules']);

    // POPs
    Route::get ('/pops',              [PopController::class, 'index']);
    Route::post ('/pops',             [PopController::class, 'store']);
    Route::get ('/pops/status',       [PopController::class, 'status']);
    Route::post('/pops/{pop}/drain',  [PopController::class, 'drain']);
    Route::post('/pops/{pop}/undrain',[PopController::class, 'undrain']);
});
```

### `DomainController.php`

```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;

class DomainController extends Controller
{
    public function store(Request $request)
    {
        $data = $request->validate([
            'domain'      => 'required|string|max:253',
            'origin'      => 'required|string',
            'origin_ssl'  => 'boolean',
            'waf'         => 'boolean',
            'signed_urls' => 'boolean',
            'cache_ttl'   => 'integer|min:60|max:31536000',
            'rate_limit'  => 'integer|min:10|max:100000',
        ]);

        $domain = strtolower($data['domain']);

        if (Redis::get("domain:{$domain}")) {
            return response()->json(['error' => 'Domain already exists'], 409);
        }

        $config = array_merge([
            'origin_ssl'  => false,
            'waf'         => false,
            'signed_urls' => false,
            'cache_ttl'   => 86400,
            'rate_limit'  => 500,
            'active'      => true,
            'created_at'  => now()->timestamp,
            'ssl_status'  => 'pending',
        ], $data, ['domain' => $domain]);

        Redis::set("domain:{$domain}", json_encode($config));
        Redis::zadd('domains:index', now()->timestamp, $domain);

        // Trigger SSL issuance (queued job)
        dispatch(new \App\Jobs\IssueSslJob($domain));

        return response()->json($config, 201);
    }

    public function update(Request $request, string $domain)
    {
        $domain = strtolower($domain);
        $raw = Redis::get("domain:{$domain}");
        if (!$raw) return response()->json(['error' => 'Not found'], 404);

        $config = json_decode($raw, true);
        $allowed = ['origin','origin_ssl','waf','signed_urls','cache_ttl','rate_limit','active'];
        foreach ($allowed as $key) {
            if ($request->has($key)) {
                $config[$key] = $request->input($key);
            }
        }
        $config['updated_at'] = now()->timestamp;

        Redis::set("domain:{$domain}", json_encode($config));

        // LRU cache on edge nodes will expire within 30s automatically
        // For immediate effect, send reload signal
        dispatch(new \App\Jobs\EdgeReloadJob($domain));

        return response()->json($config);
    }

    public function destroy(string $domain)
    {
        $domain = strtolower($domain);
        Redis::del("domain:{$domain}");
        Redis::zrem('domains:index', $domain);
        // Purge all cache for domain
        dispatch(new \App\Jobs\PurgeCacheJob($domain, 'domain'));

        return response()->json(['deleted' => true]);
    }

    public function index()
    {
        $domains = Redis::zrange('domains:index', 0, -1);
        $configs = [];
        foreach ($domains as $d) {
            $raw = Redis::get("domain:{$d}");
            if ($raw) $configs[] = json_decode($raw, true);
        }
        return response()->json($configs);
    }

    public function show(string $domain)
    {
        $raw = Redis::get("domain:" . strtolower($domain));
        if (!$raw) return response()->json(['error' => 'Not found'], 404);
        return response()->json(json_decode($raw, true));
    }
}
```

---

## 9. Step-by-Step: Add a New Site

### Prerequisites checklist

```
□ Customer's DNS is pointed to CDN edge IPs (A records)
  OR customer sets CNAME → geo-cdn.example.com
□ Origin server is accessible from CDN edge nodes
□ You are logged in:  cdnctl login
```

### Step 1 — Add domain via CLI

```bash
cdnctl domain add customer.example.com \
    --origin 52.10.20.30 \
    --waf \
    --cache-ttl=86400 \
    --rate-limit=1000
```

**Output:**
```json
{
  "domain":      "customer.example.com",
  "origin":      "52.10.20.30",
  "waf":         true,
  "active":      true,
  "ssl_status":  "pending"
}
```

### Step 2 — Verify DNS propagation

```bash
# Check that the domain resolves to your CDN edge
dig +short customer.example.com

# Expected: your edge node IP(s)
```

### Step 3 — Trigger first HTTPS request (SSL issuance)

```bash
# First request triggers Let's Encrypt HTTP-01 challenge automatically
curl -I http://customer.example.com/.cdn/health
# Should redirect to HTTPS

curl -Ik https://customer.example.com/.cdn/health
# On first hit: may see self-signed fallback cert (normal, cert issuing)
# Within 30-60 seconds: real cert is active
```

### Step 4 — Check SSL status

```bash
cdnctl ssl status customer.example.com
```

```json
{
  "domain":    "customer.example.com",
  "status":    "active",
  "issuer":    "Let's Encrypt",
  "expires":   "2025-10-15",
  "days_left": 89
}
```

### Step 5 — Test delivery

```bash
# Test cache MISS then HIT
curl -I https://customer.example.com/images/logo.png
# X-Cache: MISS

curl -I https://customer.example.com/images/logo.png
# X-Cache: HIT

# Test WAF blocking (should return 403)
curl -I "https://customer.example.com/?q=<script>alert(1)</script>"
# HTTP/2 403
```

### Step 6 — Enable signed URLs (optional)

```bash
cdnctl domain disable-feature customer.example.com --signed-urls  # open delivery (default)
# OR require signed URLs:
cdnctl domain update customer.example.com --signed-urls  # NOT yet in CLI v2, use API:
curl -X PATCH https://api.cdn.example.com/api/domains/customer.example.com \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"signed_urls":true}'

# Generate a signed URL
cdnctl sign /images/private.jpg --ttl=3600
# https://customer.example.com/images/private.jpg?expires=1735693200&signature=abc123
```

### Step 7 — Configure cache purge webhook (optional)

The customer can call the purge API directly:
```bash
# Give customer a scoped API token with purge-only permission
# Customer's deployment pipeline:
curl -X POST https://api.cdn.example.com/api/purge/prefix \
    -H "Authorization: Bearer CUSTOMER_TOKEN" \
    -d '{"domain":"customer.example.com","prefix":"/assets/"}'
```

---

## 10. Step-by-Step: Add a New POP

### When to add a POP

- Latency > 80ms from a major region
- Traffic from a region exceeds 15% of total with no local node
- Adding redundancy to an existing region

### Step 1 — Provision the VPS

```
OS:    Ubuntu 24.04 LTS
vCPU:  4+
RAM:   8 GB+
Disk:  100 GB NVMe
IP:    Static public IPv4
```

```bash
# On the new node — initial setup
apt update && apt upgrade -y
apt install -y curl jq ufw fail2ban

# Install Docker
curl -fsSL https://get.docker.com | sh
usermod -aG docker deploy

# Apply kernel tuning (copy from repo)
bash scripts/sysctl-tune.sh
```

### Step 2 — Register POP with the platform

```bash
# On your management machine
cdnctl pop add tokyo-01 103.x.x.x asia
```

### Step 3 — Join Docker Swarm

```bash
# On EU manager node — get join token
docker swarm join-token manager

# On NEW node — join swarm
docker swarm join \
    --token SWMTKN-1-xxxxx \
    185.x.x.x:2377

# Back on manager — label the node
docker node update --label-add region=asia   tokyo-01
docker node update --label-add pop=tokyo-01  tokyo-01
```

### Step 4 — Create overlay networks (auto-joined when stacks deploy)

The new node will auto-join existing overlay networks when services are scheduled there. No manual network setup needed.

### Step 5 — Deploy stacks to new node

```bash
# Services with mode: global deploy automatically to new nodes.
# If using replicated mode, force redeploy:
docker service update --force cdn_edge_openresty
docker service update --force cdn_edge_varnish
```

### Step 6 — Verify services running on new node

```bash
docker node ps tokyo-01
# Should show: openresty, varnish, node_exporter, cadvisor running
```

### Step 7 — Set up MinIO (if storage node)

```bash
# Create data directory on new node
mkdir -p /data/minio

# Deploy storage stack targeting new node:
docker stack deploy -c stacks/storage.yml cdn_storage
```

### Step 8 — Configure firewall on new node

```bash
# Run firewall setup script, adding new node IP to existing nodes' rules too
bash scripts/firewall-setup.sh

# On EU, US nodes — allow Swarm traffic from new IP
ufw allow from 103.x.x.x to any port 2377 proto tcp
ufw allow from 103.x.x.x to any port 4789 proto udp
ufw allow from 103.x.x.x to any port 7946
ufw reload
```

### Step 9 — Add to PowerDNS GeoDNS

```bash
# Edit zones/example.com.yaml — add new IP to Asia section
# Then reload PowerDNS:
docker exec $(docker ps -qf name=cdn_dns) pdns_control reload

# Or use the PowerDNS API:
curl -X PATCH http://dns.internal:8081/api/v1/servers/localhost/zones/cdn.example.com. \
    -H "X-API-Key: $PDNS_KEY" \
    -d '{"rrsets":[{"name":"cdn.example.com.","type":"A","ttl":30,"changetype":"REPLACE","records":[{"content":"185.x.x.x","disabled":false},{"content":"45.x.x.x","disabled":false},{"content":"103.x.x.x","disabled":false}]}]}'
```

### Step 10 — Undrain the POP (add to rotation)

```bash
cdnctl pop undrain tokyo-01
```

### Step 11 — Verify

```bash
cdnctl pop status

# Test from Asia with a DNS resolver in the region:
dig @8.8.8.8 cdn.example.com
# Should see new IP returned for Asia clients

# Test delivery from new POP:
curl -I --resolve cdn.example.com:443:103.x.x.x https://cdn.example.com/images/test.jpg
# X-Cache: MISS (first hit, warms cache on new POP)
```

---

## 11. Step-by-Step: Remove a Site

```bash
# 1. Disable first (returns 503, gradual off)
cdnctl domain disable customer.example.com

# 2. Wait for TTL-cached DNS (if using domain-specific DNS) — 30s
sleep 30

# 3. Purge all cache
cdnctl cache purge-domain customer.example.com

# 4. Remove domain (removes from Redis, all POPs respond 444)
cdnctl domain remove customer.example.com

# 5. Revoke any active signed URLs if needed
# (manual: collect signatures from your logs)
```

---

## 12. Improvement Ideas

### A. Wildcard & SAN certificates
For customers with subdomains (`www.customer.com`, `api.customer.com`), use DNS-01 ACME challenge via PowerDNS API to issue wildcard certs. Add a `wildcard: true` flag to the domain config.

### B. Pull-Zone vs Push-Zone
Currently all delivery is pull (fetch from origin on MISS). Add a push-zone mode where customers upload files directly to MinIO buckets via the management API, removing origin dependency entirely.

### C. Custom Cache Rules per Domain
Extend the domain config JSON:
```json
"cache_rules": [
    { "path": "/api/*",    "ttl": 0,      "bypass": true },
    { "path": "/assets/*", "ttl": 2592000, "immutable": true },
    { "path": "/images/*", "ttl": 604800 }
]
```
Lua reads these rules per request.

### D. Customer Dashboard
A simple React SPA talking to the Laravel API. Customers see their own traffic stats, manage their domain, toggle WAF, generate signed URLs — all scoped to their account.

### E. Tiered WAF
Add three levels: `off` → `monitor` (log only, no block) → `block`. The `monitor` mode lets customers safely test WAF impact before enabling hard blocking.

### F. Origin Health Checks
Background job pings each origin every 30s. If origin fails 3 consecutive checks, automatically enable `stale-if-error` mode and alert the customer. Optionally support a failover origin.

### G. Bandwidth Billing Hooks
Each edge node increments a Redis counter per domain per day (`domain:bw:customer.com:20240115`). A daily cron job reads these, generates invoices or sends webhook events to a billing system.

### H. HTTP Header Manipulation per Domain
Allow customers to add/remove response headers:
```json
"headers": {
    "add":    { "Access-Control-Allow-Origin": "*" },
    "remove": ["X-Powered-By", "Server"]
}
```

### I. GeoBlock per Domain
Optional per-domain country block list stored in Redis. Lua checks `ngx.var.geoip2_country_code` against the block list before processing.

### J. Real-time Cache Analytics
Use Varnish VSM (shared memory) + a small Go sidecar that reads varnishstat every second and pushes to Prometheus. Dashboards show per-domain cache hit ratio in near real-time.

### K. Automatic POP Bootstrap
A cloud-init or Ansible playbook that, given a new VPS IP and region, fully bootstraps the node (Docker, kernel tuning, join Swarm, deploy stacks, update firewall, add to DNS) in a single command.

---

## 13. Updated Folder Structure

```
cdn-platform/
├── scripts/
│   ├── cdnctl                     ← CLI tool (install to /usr/local/bin)
│   ├── bootstrap-pop.sh           ← New POP bootstrapper
│   ├── bootstrap-swarm.sh
│   ├── bootstrap-minio.sh
│   ├── firewall-setup.sh
│   ├── sysctl-tune.sh
│   ├── inject-secrets.sh
│   └── rotate-secrets.sh
│
├── openresty/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── conf.d/
│   │   ├── acme.conf              ← NEW: Auto-SSL config
│   │   ├── cdn.conf               ← Updated: dynamic multi-tenant
│   │   └── api.conf
│   └── lua/
│       ├── domain_router.lua      ← NEW: per-request domain config loader
│       ├── cdn/
│       │   ├── waf.lua            ← NEW: per-domain WAF
│       │   ├── signed_url.lua
│       │   ├── rate_limit.lua
│       │   └── utils.lua
│       └── init.lua
│
├── laravel-api/
│   ├── app/Http/Controllers/
│   │   ├── DomainController.php   ← NEW
│   │   ├── SslController.php      ← NEW
│   │   ├── WafController.php      ← NEW
│   │   ├── PopController.php      ← NEW
│   │   ├── PurgeController.php
│   │   ├── SignedUrlController.php
│   │   └── AnalyticsController.php
│   └── app/Jobs/
│       ├── IssueSslJob.php        ← NEW
│       ├── EdgeReloadJob.php      ← NEW
│       ├── PurgeCacheJob.php
│       └── PopHealthCheckJob.php  ← NEW
│
└── docs/
    ├── add-site.md                ← Section 9 above
    ├── add-pop.md                 ← Section 10 above
    ├── remove-site.md             ← Section 11 above
    ├── cdnctl-reference.md        ← Full CLI reference
    ├── waf-rules.md               ← WAF rule reference
    └── architecture.md
```
