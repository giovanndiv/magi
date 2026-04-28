# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Docker Compose-based self-hosted NAS/media server stack with an NGE (Neon Genesis Evangelion) theme — the repo name `magi` refers to the NERV supercomputer system. It orchestrates services for media management (*arr apps), torrenting via VPN, media playback, and various optional self-hosted tools. All services run behind a Traefik reverse proxy with TLS via Let's Encrypt (Cloudflare DNS challenge).

The NGE theme is intentional and consistent throughout — naming, comments, docs, and homepage branding all follow it. Do not treat these as placeholder names to be normalized.

| Concept | Name | NGE Reference |
|---|---|---|
| Stack | `magi` | NERV's three supercomputers |
| Machine hostname | `nerv` | NERV HQ |
| Domain | `geo-front.net` | GeoFront |
| Standard user | `shinji` | Shinji Ikari |
| Admin/root user | `gendo` | Gendo Ikari |
| Tailscale network | AT-Field | Absolute Terror Field |

Use "AT-Field" when referring to Tailscale in comments, commit messages, and documentation.

## Deployment Context

- **Host**: HP EliteDesk 800 G4, i7-8700, 16GB RAM, headless Debian
- **Machine hostname**: `nerv` (prompt: `shinji@nerv:~$`)
- **Stack name**: `magi`
- **Domain**: `geo-front.net`
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

All containers share the `docker-compose-nas` Docker network. Traefik routes all external traffic via subdomain routing — each service gets its own subdomain (e.g., `sonarr.geo-front.net`, `radarr.geo-front.net`). Homepage is served at `magi.geo-front.net`. A wildcard TLS certificate (`*.geo-front.net`) is issued via the Cloudflare DNS challenge so all subdomains share a single cert.

**qBittorrent is special**: it runs on the `vpn` container's network (`network_mode: "service:vpn"`), so it can only reach the internet through the AirVPN WireGuard VPN via Gluetun. The VPN container must be healthy before qBittorrent starts. Homepage widgets for qBittorrent use `http://vpn:8080` (not `http://qbittorrent:8080`).

### Active Services

All services accessible via subdomain routing on `geo-front.net`:

| Service | URL | Notes |
|---|---|---|
| Homepage | magi.geo-front.net | Main dashboard |
| Sonarr | sonarr.geo-front.net | TV show management |
| Sonarr Anime | sonarr-anime.geo-front.net | Anime series management |
| Radarr | radarr.geo-front.net | Movie management |
| Radarr Anime | radarr-anime.geo-front.net | Anime movie management |
| Prowlarr | prowlarr.geo-front.net | Indexer management |
| qBittorrent | qbittorrent.geo-front.net | Torrent client (via VPN) |
| Jellyfin | jellyfin.geo-front.net | Media server |
| Bazarr | bazarr.geo-front.net | Subtitle management |
| Seerr | seerr.geo-front.net | Media requests |
| Autobrr | autobrr.geo-front.net | Torrent automation |
| Vaultwarden | vaultwarden.geo-front.net | Password manager |
| AdGuard Home | dns.geo-front.net | DNS and ad blocking |
| Recyclarr | (no UI) | Quality profile sync |
| Flaresolverr/Byparr | (internal only) | Cloudflare bypass, port 8191 |

### Optional Services via Profiles

Some services in `docker-compose.yml` require enabling via `COMPOSE_PROFILES`:
`lidarr`, `sabnzbd`, `flaresolverr`, `calibre-web`, `suggestarr`, `cross-seed`, `cleanuparr`

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
- `HOSTNAME=magi.geo-front.net` — used for Traefik routing and TLS certs
- `COMPOSE_FILE` — colon-separated list of compose files to load
- `COMPOSE_PROFILES` — comma-separated optional profile names to enable

## Post-Deploy Manual Steps

Steps that cannot be automated and must be done by hand after first `docker compose up -d`.

### 1. Run update-config.sh

After the stack is up and all *arr apps have initialized their `config.xml` files:

```bash
./update-config.sh
```

This populates API keys in `.env` and sets URL bases. Run it before configuring Homepage widgets or any service that reads from `.env`.

### 2. Jellyfin — Hardware Acceleration & Media Libraries

**Hardware acceleration:**
1. Dashboard → Playback → Transcoding
2. Set Hardware acceleration to **Intel QuickSync (QSV)**
3. Set VA-API device to `/dev/dri/renderD128`
4. Enable all codec checkboxes (H.264, HEVC, MPEG-2, VC-1, VP8, VP9, AV1, etc.)
5. Save

**Media libraries** — add one library per folder:
| Library type | Path |
|---|---|
| Movies | `/data/media/movies` |
| TV Shows | `/data/media/tv` |
| Anime (TV Shows) | `/data/media/anime` |
| Music | `/data/media/music` |

### 3. Seerr — Pre-create Config Directory

Overseerr/Jellyseerr requires the logs directory to exist with the correct ownership before first start, otherwise it fails to write logs and may crash:

```bash
sudo mkdir -p ./seerr/logs
sudo chown -R 1000:1000 ./seerr
```

Restart the container after creating the directory if it was already started.

### 4. AdGuard Home — Setup Wizard

On first access, AdGuard Home presents a setup wizard:
1. Complete the wizard (set admin credentials, choose listen interfaces)
2. After setup, navigate to Filters → DNS blocklists and add desired blocklists
3. After migrating to subdomain routing, the DNS rewrite must be a wildcard entry rather than a single root-domain entry. In Filters → DNS rewrites, add `*.geo-front.net` → LAN IP so that all service subdomains resolve correctly on the local network instead of routing over the AT-Field.

### 5. Samba — Set gendo's Samba Password

The Ansible playbook installs Samba and configures the `[geofront]` share, but cannot set the Samba password non-interactively without storing it in plaintext. After provisioning, set it manually:

```bash
sudo smbpasswd -a gendo
```

The share is accessible over the AT-Field (tailscale0) only. Connect from a client using:

```
\\<tailscale-ip>\geofront
```

or (if DNS resolves via AdGuard Home):

```
\\nerv\geofront
```
