# CDN Platform

A production-grade, self-hosted mini CDN built on Docker Swarm — inspired by Cloudflare, BunnyCDN, and KeyCDN.

---

## Documentation

| Document | Description |
|---|---|
| [CDN Platform v1](./docs/v1-core-infrastructure.md) | Core infrastructure: architecture, Docker Swarm stacks, OpenResty, Varnish VCL, Redis, MinIO, PowerDNS, observability, CI/CD, security hardening, backup & disaster recovery |
| [CDN Platform v2](./docs/v2-multitenant.md) | Multi-tenant extensions: automatic SSL, per-domain WAF, `cdnctl` CLI, domain registry, step-by-step guides for adding sites and POPs |

---

## Stack

| Component | Role |
|---|---|
| **OpenResty** | HTTPS termination, Lua business logic, signed URL validation, rate limiting, WAF |
| **Varnish** | Edge cache, grace mode, stale-if-error, Range request support |
| **Redis** | Domain registry, SSL cert store, rate limiting, signed URL revocations |
| **MinIO** | S3-compatible origin object storage |
| **PowerDNS** | Authoritative DNS with GeoDNS routing |
| **Prometheus + Grafana + Loki** | Metrics, dashboards, log aggregation |
| **Docker Swarm** | Container orchestration across 3 regions |

---

## Regions

```
EU node   → 185.x.x.x
US node   → 45.x.x.x
Asia node → 103.x.x.x
```

---

## Quick Start

```bash
# 1. Clone and configure
cp .env.example .env

# 2. Bootstrap Swarm (run on first manager)
bash scripts/bootstrap-swarm.sh

# 3. Inject secrets
bash scripts/inject-secrets.sh

# 4. Deploy all stacks
bash scripts/deploy-all.sh

# 5. Install CLI
cp scripts/cdnctl /usr/local/bin/cdnctl && chmod +x /usr/local/bin/cdnctl

# 6. Login and add your first site
cdnctl login
cdnctl domain add customer.com --origin 1.2.3.4 --waf
```

---

## License

MIT
