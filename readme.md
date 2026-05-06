* Nextcloud config for docker compose with reverse proxy

# Docker Compose — Nextcloud + Static Site with Reverse Proxy

## Overview

This project runs:

- **Nextcloud** (with MariaDB + Redis)
- **A static website** (built from `./simple_static_webpage`)
- **A reverse proxy** handling TLS termination and routing

Two compose files are provided:

| File | Proxy |
|------|-------|
| `compose.yaml` | Nginx Proxy Manager (original) |
| `compose_new.yaml` | Traefik v3.4 (replacement) |

---

## Migration: Nginx Proxy Manager → Traefik

### What changed

| Area | Before (`compose.yaml`) | After (`compose_new.yaml`) |
|------|--------------------------|----------------------------|
| Proxy service | `npm` — jc21/nginx-proxy-manager:latest | `traefik` — traefik:v3.4 |
| Proxy config | GUI-based (port 81 admin UI) | Label-based on each service |
| TLS certificates | Managed via NPM UI, stored in `./proxy_letsencrypt` | Automatic via ACME HTTP challenge, stored in `./traefik_letsencrypt/acme.json` |
| HTTP → HTTPS redirect | Configured per-host in NPM UI | Global entrypoint redirect |
| Nextcloud `TRUSTED_PROXIES` | `npm` | `traefik` |
| Exposed ports | 80, 81 (admin), 443 | 127.0.0.1:80, 127.0.0.1:443, 127.0.0.1:8082 (all localhost only) |
| Dashboard | NPM admin UI on port 81 | No external domain — local access only |

### Routing rules (via Docker labels)

| Service | Domain | Notes |
|---------|--------|-------|
| `app` (Nextcloud) | `nextcloud.kgune.com` | HSTS headers middleware included |
| `static-site` | `kgune.com` | Adjust host if using a subdomain |
| `traefik` dashboard | `http://localhost:8082` | Local only, enabled via `--api.insecure=true` |

### Volumes

| Volume / Bind Mount | Purpose |
|---------------------|---------|
| `./mariadb_volume` | MariaDB data |
| `./nextcloud_data` | Nextcloud user data |
| `./traefik_letsencrypt` | ACME certificate storage |
| `/var/run/docker.sock` (read-only) | Traefik Docker provider |

---

## Pre-deployment checklist

1. **DNS** — Ensure `nextcloud.kgune.com` and `kgune.com` resolve to your server (or to whatever tunnel/proxy sits in front of localhost).
2. **Static site port** — The `static-site` container must expose port 80 internally (check its Dockerfile).
3. **Dashboard access** — The Traefik dashboard is enabled locally at `http://localhost:8082` (via `--api.insecure=true`). It is not exposed externally.
4. **Environment files** — `db.env` must exist with `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, `MYSQL_DATABASE`, and `MYSQL_USER`.
5. **First run** — On first start, Nextcloud auto-creates the admin account using the credentials in the compose file. Change `NEXTCLOUD_ADMIN_PASSWORD` before deploying.

---

## Usage

```bash
# Start with Traefik proxy
docker compose -f compose_new.yaml up -d

# View logs
docker compose -f compose_new.yaml logs -f traefik

# Stop everything
docker compose -f compose_new.yaml down
```

---

## Network topology

```
Internet
  │
  ▼
┌──────────┐
│ Traefik  │  :80 (redirect) / :443 (TLS)
└────┬─────┘
     │  proxy-tier network
     ├──────────────► app (Nextcloud :80)
     └──────────────► static-site (:80)

     default network
     app ──► db (MariaDB :3306)
     app ──► redis (:6379)
```


# Further Details

* Here's what changed in compose_new.yaml:

    - Replaced Nginx Proxy Manager with Traefik v3.4:
        - Removed the npm service entirely
        - Added a traefik service with Docker provider, automatic HTTPS via Let's Encrypt (HTTP challenge), and a global HTTP→HTTPS redirect

    - Routing via labels (no separate config files needed):
        - app (Nextcloud) → routed at nextcloud.kgune.com with TLS + HSTS headers middleware
        - static-site → routed at kgune.com with TLS (adjust the host if your static site uses a different domain)
        - Traefik dashboard → accessible locally at http://localhost:8082 (not exposed externally)

    - Other adjustments:
        - TRUSTED_PROXIES in the Nextcloud app changed from npm to traefik
        - Certificates stored in acme.json instead of the old NPM Let's Encrypt volume
        - Port 81 (NPM admin UI) is gone — Traefik's dashboard is served over HTTPS on the same port

    - Things to verify before deploying:
        - Update the Host() rule on static-site if your website uses a subdomain like www.kgune.com
        - If you want to protect the Traefik dashboard, add a basic-auth middleware or remove those dashboard labels
        - Make sure DNS for nextcloud.kgune.com and kgune.com point to your server (or to the tunnel/proxy in front of localhost)