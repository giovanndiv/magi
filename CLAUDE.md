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

## Working Style & Preferences

**Before writing any Docker Compose service, configuration, or CLI commands:**
- Always consult official documentation first. Do not guess at syntax, image names, environment variables, or CLI flags.
- For Recyclarr: https://recyclarr.dev
- For Docker Compose services: check the official image documentation on Docker Hub or the project's GitHub
- For CLI tools: run --help or check official docs before suggesting commands
- If unsure about a command's syntax, say so explicitly rather than guessing

**Git workflow — two patterns depending on importance:**

For important changes (compose changes, config changes, anything that affects running services):
```bash
git add . && git commit -m "your message" && git push origin branch-name && gh pr create --base master --head branch-name --title "your title" && gh pr comment --body "@claude review"
```
After creating the PR, poll until the GitHub Claude bot finishes its review:

while true; do
  COMMENT=$(gh pr view <PR_NUMBER> --json comments --jq '.comments[-1].body')
  if echo "$COMMENT" | grep -q "Claude finished"; then
    echo "Review complete:"
    echo "$COMMENT"
    echo "--- Stopping. ---"
    break
  fi
  sleep 30
done

After the loop breaks, stop completely. Print the review comment, list each issue as a numbered item, and wait for the user to tell you which ones to fix before doing anything. Address any issues, force push, then merge:
```bash
gh pr merge --merge && git checkout master && git pull origin master && git branch -d branch-name && git push origin --delete branch-name
```
Then on server: `cd ~/magi && git pull origin master`

For unimportant changes (docs only, minor fixes):
```bash
git add . && git commit -m "your message" && git push origin branch-name && gh pr create --base master --head branch-name --title "your title" && gh pr merge --merge && git checkout master && git pull origin master && git branch -d branch-name && git push origin --delete branch-name
```
Then on server: `cd ~/magi && git pull origin master`

**Architect/developer split**: Claude Chat acts as architect and decision-maker. Claude Code is the executor. When Claude Chat generates a prompt for Claude Code, always include: read CLAUDE.md first, exact changes to make, which git workflow to use, branch name, and PR title.

**Healthchecks**: Always use wget instead of curl — curl is often not available in minimal images. Always include timeout and start_period. Check cleanuparr as the reference pattern.

**Env vars and container labels**: After adding a new variable to .env on the server, any container using that variable in labels needs --force-recreate, not just restart. If compose config fails with a missing file error, fix the missing file first — it breaks variable resolution globally.

**Always use `gh pr create --base master --head branch-name`** — never use the GitHub web UI for PRs as it defaults to the upstream repo base.

**When you need file contents from the user**, provide the exact command to get them. Examples:
- Need CLAUDE.md: `cat ~/magi/CLAUDE.md | clip.exe`
- Need docker-compose.yml: `cat ~/magi/docker-compose.yml | clip.exe`
- Need git diff: `git diff | clip.exe`
- Need recyclarr config: `cat ~/magi/recyclarr/configs/recyclarr.yml | clip.exe`
Never assume file contents — always ask and provide the command.

**When showing compose or config changes:** always show the complete updated block, not just the diff or a description of what changed.

**When something isn't working:** look up the official docs or error message before suggesting fixes. Acknowledge uncertainty explicitly rather than guessing. Never let the user run a command you are not confident about.

**Communication:**
- When wrong or guessing, say so immediately
- Do not repeat yourself — if the user confirms something is done, move on
- Do not over-explain decisions the user has already made
- When the user implies they want speed, skip hand-holding
- Give clear recommendations when asked "should I do X or Y" — not "it depends"
- Don't suggest backlog items as new ideas

**Session management:**
- At the end of every session, remind the user to update docs/backlog/CLAUDE.md and offer to generate the update prompt
- Summarize what was completed and what is still pending at wrap-up
- When the conversation becomes long and slow to respond, proactively suggest starting a new session and provide the cat commands needed to bring the new session up to speed

**Config and file changes:**
- When editing recyclarr.yml, docker-compose.yml, or any config file, always ask for the current contents first before generating changes
- Never generate a partial config — always show the complete file or block
- When switching tools (e.g. Recyclarr → Profilarr), clearly state upfront what needs to be cleaned up from the old tool before setting up the new one

**Claude Code prompt format**: write all prompts as plain text with no markdown code fences (no triple backticks). Yaml and bash blocks go inline with plain indentation. No nested formatting.

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

All containers share the `nerv` Docker network. Traefik routes all external traffic via subdomain routing — each service gets its own subdomain (e.g., `sonarr.geo-front.net`, `radarr.geo-front.net`). Homepage is served at `magi.geo-front.net`. A wildcard TLS certificate (`*.geo-front.net`) is issued via the Cloudflare DNS challenge so all subdomains share a single cert.

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
| Profilarr | profilarr.geo-front.net | Quality profile management (replacing Recyclarr) |
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
2. Clears the URL base in each *arr app's `config.xml` (originally SET URL bases; updated after migrating to subdomain routing to CLEAR them instead — do not re-add URL base setting logic)
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
