# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Docker Compose-based self-hosted NAS/media server stack with an NGE (Neon Genesis Evangelion) theme — the repo name `magi` refers to the NERV supercomputer system. It orchestrates services for media management (*arr apps), torrenting via VPN, media playback, and various optional self-hosted tools. All services run behind a Traefik reverse proxy with TLS via Let's Encrypt (Cloudflare DNS challenge).

The NGE theme is intentional and consistent throughout — naming, comments, docs, and homepage branding all follow it. Do not treat these as placeholder names to be normalized.

| Concept | Name | NGE Reference |
|---|---|---|
| Stack | `magi` | NERV's three supercomputers |
| Machine hostname | `nerv` | NERV HQ |
| Domain | `geofront.com` | GeoFront (placeholder) |
| Standard user | `shinji` | Shinji Ikari |
| Admin/root user | `gendo` | Gendo Ikari |
| Tailscale network | AT-Field | Absolute Terror Field |

Use "AT-Field" when referring to Tailscale in comments, commit messages, and documentation.

## Deployment Context

- **Host**: HP EliteDesk 800 G4, i7-8700, 16GB RAM, headless Debian
- **Machine hostname**: `nerv` (prompt: `shinji@nerv:~$`)
- **Stack name**: `magi`
- **Domain**: `geofront.com` (placeholder — update `.env` when finalized)
- **Remote access**: Tailscale, referred to as the **AT-Field** in comments and docs (A record points to Tailscale IP `100.x.x.x`; AdGuard Home rewrites to LAN IP for local access)
- **Users**: `shinji` (standard, runs services), `gendo` (admin/sudo)
- **Storage**: 2×14TB drives pooled with `mergerfs` + `snapraid` (~28TB usable with parity); mounted at `/mnt/data`
- **Provisioning**: Ansible playbook under `ansible/` handles host setup (packages, users, mounts, Docker, Tailscale/AT-Field)

## Commands

**Start all services:**
```bash
docker compose up -d
```

**Start with specific optional profiles:**
```bash
COMPOSE_PROFILES=lidarr,sabnzbd docker compose up -d
```

**View logs for a service:**
```bash
docker compose logs -f <service>
```

**Validate docker-compose config (used in CI):**
```bash
cp .env.example .env
docker compose config
```

**First-time setup** — update base URLs and extract API keys into `.env`:
```bash
./update-config.sh
```

**Update permissions for DATA_ROOT** (per TRaSH guides):
```bash
source .env
sudo chown -R shinji:shinji $DATA_ROOT
sudo chmod -R a=,a+rX,u+w,g+w $DATA_ROOT
```

## Architecture

### Compose File Structure

The main `docker-compose.yml` defines core services. Optional services live in subdirectory compose files that are loaded via `COMPOSE_FILE` in `.env`:

- `adguardhome/docker-compose.yml`
- `immich/docker-compose.yml`
- `joplin/docker-compose.yml`
- `tandoor/docker-compose.yml`
- `homeassistant/docker-compose.yml`
- `vaultwarden/docker-compose.yml`
- `privoxy/docker-compose.yml`
- `paperless/docker-compose.yml`

Each subdirectory also has its own `README.md` with service-specific notes.

### Networking

All containers share the `docker-compose-nas` Docker network. Traefik routes all external traffic via path prefixes (e.g., `/sonarr`, `/radarr`) or dedicated hostnames (e.g., `$SEERR_HOSTNAME`, `$ADGUARD_HOSTNAME`).

**qBittorrent is special**: it runs on the `vpn` container's network (`network_mode: "service:vpn"`), so it can only reach the internet through the PIA WireGuard VPN. The VPN container must be healthy before qBittorrent starts. Homepage widgets for qBittorrent use `http://vpn:8080` (not `http://qbittorrent:8080`).

### Optional Services via Profiles

Some services in `docker-compose.yml` require enabling via `COMPOSE_PROFILES`:
`lidarr`, `sabnzbd`, `flaresolverr`, `calibre-web`, `autobrr`, `suggestarr`, `cross-seed`, `cleanuparr`

### Hardware-Specific Overrides

All hardware-specific changes (Intel UHD 630 VA-API devices for Jellyfin, group_add for render access, etc.) go in `docker-compose.override.yml`. Never modify `docker-compose.yml` directly for hardware concerns. Add the override to `COMPOSE_FILE` in `.env`:

```env
COMPOSE_FILE=docker-compose.yml:...:docker-compose.override.yml
```

### Storage

The two 14TB drives are pooled with `mergerfs` + `snapraid`, mounted at `/mnt/data`. This gives ~28TB usable with parity (not mirroring). `snapraid` parity syncs are scheduled — data added since the last sync is unprotected until the next sync runs.

### Configuration Pattern

- All service configs are stored under `$CONFIG_ROOT` (default: `.`), e.g., `./sonarr/`, `./radarr/`
- Media/data lives under `$DATA_ROOT` (`/mnt/data`)
- Downloads land in `$DOWNLOAD_ROOT` (`/mnt/data/torrents`)
- Hardlinks are used between download and media folders to avoid double storage — this requires `$DOWNLOAD_ROOT` to be a subdirectory of `$DATA_ROOT` and both mounted under `/data` inside containers

### Data Volume Layout

```
/data
├── torrents/       ← $DOWNLOAD_ROOT, qBittorrent writes here
│   ├── movies/
│   └── tv/
└── media/          ← Sonarr/Radarr move files here via hardlink
    ├── movies/
    ├── tv/
    └── music/
```

### Homepage Integration

Services expose themselves to Homepage via Docker labels (`homepage.group`, `homepage.name`, `homepage.widget.*`). API keys for widgets come from `.env` variables and are passed as label values.

### update-config.sh

This script automates first-run configuration:
1. Reads API keys from each *arr app's `config.xml` and writes them to `.env`
2. Sets the URL base in each *arr app's `config.xml`
3. Sets qBittorrent's WebUI password to `adminadmin`
4. Restarts affected containers

### Environment

All configuration is in `.env` (copy from `.env.example`). Key variables:
- `USER_ID` / `GROUP_ID` — should match `shinji`'s UID/GID on the host (`id shinji`)
- `HOSTNAME=magi.geofront.com` — used for Traefik routing and TLS certs (domain is a placeholder until finalized)
- `COMPOSE_FILE` — colon-separated list of compose files to load
- `COMPOSE_PROFILES` — comma-separated optional profile names to enable
