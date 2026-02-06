# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dockerized homelab / NAS / media server setup. A single `docker-compose.yml` orchestrates ~30 services for media acquisition, management, serving, monitoring, and infrastructure. No build system, CI, or tests — deployment is entirely docker-compose based.

## Common Commands

```bash
# Pull latest images and restart changed containers
docker compose pull
docker compose up -d

# Restart a single service
docker compose restart <service>

# Force recreate (needed when only mounted file contents change, not compose config)
docker compose up -d --force-recreate <service>

# Rebuild custom-built services (homepage)
docker compose build homepage
docker compose up -d homepage

# View logs
docker compose logs -f <service>

# Check health status
docker compose ps
```

## Architecture

### docker-compose.yml Structure

Uses YAML anchors for shared configuration:
- `x-env` — common env vars (PUID=1000, PGID=1000, TZ=Europe/London)
- `x-healthcheck` — default health check (curl icanhazip.com)
- `x-common` — combines env, healthcheck, restart policy, network, and `no-new-privileges:true`

Services reference these via `<<: *common`. Some services override specific fields (e.g., custom healthcheck tests).

### Service Dependency Chain

```
flaresolverr → prowlarr → [sonarr, radarr, lidarr, readarr] → autobrr → omegabrr
                              ↑ also depend on qbittorrent + sabnzbd
cross-seed depends on prowlarr + qbittorrent
qbit_manage depends on qbittorrent
```

All `depends_on` use `condition: service_healthy`.

### Networking

- Custom bridge network `home` for most services
- `network_mode: host` for glances, plex, netdata
- Traefik reverse proxy with Cloudflare DNS challenge for `*.petalas.dev`
- qBittorrent has built-in VPN (PIA/WireGuard) with `NET_ADMIN` capability

### Data Layout

- `/mnt/storage/` — media root (tv, movies, music, books, cross-seeds)
- `./config/<service>/` — per-service persistent config
- `./scripts/` — webhook and init scripts (xseed.sh for *arr services, mam-seedbox-update for qbittorrent)
- `./dockerfiles/` — custom Dockerfiles and source for builds
- `./images/` — custom dashboard images
- `.env` — all secrets/API keys (not versioned)

### Custom Homepage Build

Homepage dashboard is built from source (not the registry image) via `dockerfiles/homepage`. It clones the upstream repo and replaces widget components (sabnzbd.jsx, qbittorrent.jsx) with custom versions supporting `refreshInterval`, plus a custom theme/color system.

### Homepage Widget Integration

Services expose dashboard widgets via Docker labels:
```yaml
- homepage.group=media
- homepage.widget.type=sonarr
- homepage.widget.url=http://192.168.1.102:32784
- homepage.widget.key=${SONARR_API_KEY}
```

### Cross-Seed Webhook Flow

`scripts/xseed.sh` is mounted into *arr containers at `/scripts`. Sonarr/Radarr/Lidarr call it on download completion. It deduplicates via a log file and POSTs to the cross-seed API, handling both torrent (qBittorrent) and usenet (SABnzbd) clients differently.

### MAM Dynamic Seedbox IP Update

`scripts/mam-seedbox-update` is mounted into the qbittorrent container as an s6 init script (`/etc/cont-init.d/99-mam-seedbox-update`). It keeps the MyAnonamouse dynamic seedbox IP in sync with the VPN IP.

How it works:
1. Writes a small updater script to `/config/mam-update.sh` (used by cron)
2. Installs an hourly crontab entry and starts `crond`
3. Backgrounds a subshell that waits for the WireGuard tunnel (`wg0` interface UP), then runs the initial MAM update

Key details:
- Uses `#!/command/with-contenv bash` so `MAM_ID` is resolved from the container environment (set via `.env`)
- **Must wait for `wg0` UP** — not just internet connectivity. Before VPN firewall rules are applied, outbound traffic leaks through the host network, so checking `icanhazip.com` alone returns the host IP, not the VPN IP
- First call uses `-b "mam_id=..."` to bootstrap a cookie jar at `/config/mam.cookies`; subsequent hourly calls use the cookie jar (`-b` and `-c` pointing at the jar file), per MAM's official docs
- Logs to `/config/mam-seedbox.log` with timestamps and HTTP status codes
- 5-minute timeout if VPN never comes up

Files generated at runtime inside the container (persisted in `./config/qbittorrent/`):
- `mam.cookies` — curl cookie jar
- `mam-seedbox.log` — update log
- `mam-update.sh` — cron-called updater script

## Key Files

- `docker-compose.yml` — the entire stack definition (~750 lines)
- `.env` — secrets, API keys, credentials (never commit)
- `.gitignore` — complex exclusion pattern; most config dirs are ignored except specific template files (recyclarr.yml, config.js, config.yml, config.toml, etc.)
- `config/traefik/traefik.yaml` — Traefik routing/TLS config
- `config/homepage/` — dashboard YAML configs (services, widgets, bookmarks)
- `scripts/mam-seedbox-update` — s6 init script for MAM dynamic seedbox IP updates (mounted into qbittorrent)

## Version Pinning

Most services use `latest` or branch tags (`:develop`). Readarr is pinned to `0.4.18-develop` because it is deprecated. qBittorrent uses explicit release tags (e.g., `release-5.1.4`).
