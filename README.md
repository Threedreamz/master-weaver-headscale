# master-weaver-headscale

Self-hosted [Headscale](https://headscale.net) control-plane for the threedreamz.com VPN mesh. Provides WireGuard-based overlay networking across all ecosystem services, Hetzner VPSes, and developer machines.

## Overview

- **Control plane:** Headscale v0.23.0 (open-source Tailscale control server)
- **Domain:** `headscale.threedreamz.com`
- **Auth:** FinderAuth OIDC (`https://auth.finderfinder.org`)
- **DNS magic:** `ts.threedreamz.com` base domain with MagicDNS

## Architecture (co-locate variant)

This repo uses the **co-locate variant**: Headscale runs on a shared VM that already runs Caddy (the `deploy hetz server` at `157.90.122.114`, also serving `deploy.ersatzteildrucken.org`). There is **no in-stack Caddy** — the host Caddy handles TLS for `headscale.threedreamz.com` and reverse-proxies to `127.0.0.1:8080`.

```
Internet
   │
   ▼
Host Caddy :443 (Let's Encrypt TLS)   ← headscale.threedreamz.com
   │
   ▼  reverse_proxy 127.0.0.1:8080   (localhost, not docker network)
   │
Headscale :8080 (bound to 127.0.0.1 only)
   │
   ├─ SQLite → /data/headscale/db.sqlite3
   ├─ WireGuard keys → /data/headscale/noise_private.key
   └─ OIDC secret → /etc/headscale/oidc-secret (host path, Phase 4)
```

STUN/DERP traffic flows directly via `0.0.0.0:3478/udp` (published on host, used by WireGuard peers).

**Host Caddy config snippet** (add to the existing Caddyfile on `deploy hetz server`):

```
headscale.threedreamz.com {
    reverse_proxy 127.0.0.1:8080
}
```

## Deploy

Full provisioning is handled by the GMW provisioning script:

```bash
node ../../scripts/provision-hetzner-headscale.cjs --confirm
```

Manual bring-up on an already-provisioned host:

```bash
# 1. Ensure /etc/headscale/oidc-secret exists on the host (Phase 4 populates this)
# 2. Ensure /data/headscale dir exists with correct permissions
sudo mkdir -p /data/headscale
# 3. Start the stack (no Caddy container — host Caddy handles TLS)
docker compose -f infra/docker-compose.yml up -d
# 4. Add headscale.threedreamz.com block to host Caddy config, then reload:
#    caddy reload --config /etc/caddy/Caddyfile
```

## Manage

```bash
# List nodes
docker compose -f infra/docker-compose.yml exec headscale headscale nodes list

# Create a user (namespace)
docker compose -f infra/docker-compose.yml exec headscale headscale users create <name>

# Generate pre-auth key
docker compose -f infra/docker-compose.yml exec headscale headscale preauthkeys create --user <name> --expiration 24h

# Show routes
docker compose -f infra/docker-compose.yml exec headscale headscale routes list
```

## Backup

Cron at 03:00 daily on the host:

```bash
0 3 * * * mkdir -p /data/backups/$(date +\%F) && \
  cp /data/headscale/db.sqlite3 /data/backups/$(date +\%F)/db.sqlite3 && \
  cp /data/headscale/noise_private.key /data/backups/$(date +\%F)/noise_private.key
```

Both `db.sqlite3` (all nodes, routes, policies) and `noise_private.key` (WireGuard identity — loss requires all clients to re-enroll) must be backed up together.

## Reference

- Headscale docs: https://headscale.net
- Config schema: https://github.com/juanfont/headscale/blob/v0.23.0/config-example.yaml
- GMW plan: `.claude/plans/radiant-knitting-adleman.md`
- Provisioning script: `scripts/provision-hetzner-headscale.cjs`
