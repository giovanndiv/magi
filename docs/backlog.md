# MAGI Project Backlog

## Pending Tasks

### Infrastructure
- AUTOBRR_HOSTNAME: already added to .env.example ✓
- VAULTWARDEN_HOSTNAME: already added to .env.example ✓
- Rename docker network from `docker-compose-nas` to `nerv` in docker-compose.yml
  and adguardhome/docker-compose.yml — done ✔
- Add recyclarr to docker-compose.yml (image: ghcr.io/recyclarr/recyclarr:8, 
  user 1000:1000, RECYCLARR_CREATE_CONFIG=true)
- Daily VPN restart cron: added via crontab -e as gendo (0 4 * * * docker restart vpn) ✓
- Set up snapraid alongside mergerfs for parity protection on the 2x14TB drives
- Get a second SSD for container config storage (current 128GB M.2 is OS + configs)
- Consider upgrading RAM from 16GB if running LLM (Ollama) in future
- Autobrr IRC announce: configure DC and SP IRC announce in autobrr Settings → IRC. Then create filters routing TV/anime/movies to correct arr instances.
- qBittorrent seeding rules: verify minimum seed time and ratio are set correctly for each private tracker category to avoid HnR violations on DC (5 days / 1.0) and SP (10 days / 1.0).
- LVM restructure: reformat both 14TB drives as a single LVM logical volume for true 28TB single filesystem. Required for native hardlink support without workarounds. Schedule during maintenance window when no active seeding obligations. Process: back up media, wipe drives, create LVM volume, restore media, git clone stack, docker compose up. Current workaround (media moved to disk2) provides hardlinks within 14TB cap.
- Replace copies with hardlinks: run `hardlink -v /mnt/data/torrents /mnt/data/media` to convert existing duplicate files to hardlinks. Already run on May 20 2026 for existing library. Re-run after any bulk imports.

### Services To Add
- **Cross-seed**: deferred until library grows. Already in docker-compose.yml behind profile. Enable with
  `COMPOSE_PROFILES=cross-seed`. Automates cross-seeding torrents across trackers for better
  availability. Needs configuration at `~/magi/cross-seed/config.js`.
- **Cross-seed daemon mode**: the recovery run used a one-shot `dataDirs` search config. Strip `dataDirs` and switch cross-seed to daemon mode for ongoing cross-seeding of new downloads. When doing so, watch for injected torrents landing stopped/"done" and not announcing.
- Immich: photo management, add when ready (note: needs ~3-4GB RAM, heaviest service)
- Ollama + Open WebUI: local LLM, CPU only at 16GB RAM, expect ~5-10 tok/sec
- Flaresolverr: replaced with Byparr (ghcr.io/thephaseless/byparr:latest), running on port 8191 ✓
- Cleanuparr: add to compose for automated queue cleanup (removes stalled/malicious
  downloads, triggers re-search). Image: ghcr.io/cleanuparr/cleanuparr:latest. Has web UI on port 11011.
- Maintainerr: add to compose for long-term library management (removes unwatched
  content to free disk space). Add when library has significant content.
- Anime autobrr filters: add Anime TV → Sonarr-anime and Anime Movies → Radarr-anime
  filters in autobrr when ready
- Jellyfin anime plugins: Install AniDB and AniList plugins from Jellyfin plugin catalog
  for accurate anime metadata. Configure in anime library settings after installing.

#### Done
- **Autobrr**: deployed, IRC connected, filters configured (Freeleech, TV→Sonarr, Movies→Radarr) ✓
- **Vaultwarden**: deployed, signups disabled, admin panel enabled, Bitwarden extension connected ✓
- **Profilarr**: deployed, Dumpstarr database linked, all four arr instances connected, profiles synced ✔
- **Dozzle**: deployed (amir20/dozzle:latest, v10.6.5) via the ADD_SERVICE playbook. Container log viewer at dozzle.geo-front.net, gated behind the `dozzle` compose profile (now in server COMPOSE_PROFILES so it starts with the stack). Read-only docker socket, native `/dozzle healthcheck` (no shell/wget in image), simple file-based auth enabled (DOZZLE_AUTH_PROVIDER=simple, user `gendo`, hashed users.yml at ~/magi/dozzle/users.yml — server-only, `/dozzle/` gitignored). Login verified (correct creds → 200+JWT, bad creds → 401). ✓

### Configuration
- Vaultwarden backup: configured rclone-backup with Google Drive (RcloneBackup remote), daily 2am cron, 30 day retention, zip encrypted. Backup verified working. ✔
- DC: added to Prowlarr with general tag, seed ratio 1.0, seed time 7200 minutes. IRC announce setup pending in autobrr.
- SP: added to Prowlarr with general tag, seed ratio 1.0, seed time 14400 minutes. IRC announce setup pending in autobrr.
- Autobrr filters for SP and DC: pending — need IRC announce configured and filters created for TV, anime, movies routing
- Profilarr v2 migration: pending — v2 released May 2026, not compatible with v1. Wait for stability before migrating. New image will be ghcr.io/dictionarry-hub/profilarr:latest
- Gluetun control server auth: pending — Gluetun will require auth on /v1/vpn/status in v3.40. Configure before upgrading.
- Quality Definitions: set min/max values from TRaSH Guides in all four arr instances. One-time manual setup.
- Radarr upgrade cutoff: set Upgrades Allowed cutoff in Movies 1080p profile to prevent unwanted upgrades affecting private tracker ratios.
- Unpackerr: add Sonarr Anime and Radarr Anime instances to docker-compose.yml environment variables.
- First real run of CONFIGURE_SERVICE: use it on the next genuinely config-driven service (e.g. a Recyclarr/Profilarr sync config) as its validation-and-hardening pass.
- Bazarr: fixed — was using opensubtitles.org credentials instead of opensubtitles.com. Account created on .com, authentication now working.
- Sonarr Anime: when adding series, always set Series Type to Anime (not Standard). Also set default Series Type to Anime in Seerr → Settings → Sonarr Anime instance.
- Jellyfin: verify QuickSync QSV is actually being used during transcoding 
  (play content and check Dashboard > Activity)
- Jellyfin anime plugins: AniDB, AniList, and TheTVDB plugins installed and configured. Anime library metadata downloaders set to AniDB > AniList > TheTVDB > TheMovieDb. Image fetchers same order. AniDB enabled for Seasons and Episodes.
- Seerr anime metadata provider: changed from TMDB to TheTVDB for anime. Series stays as TMDB.
- Cross-seed: config directory created at ~/magi/cross-seed/. Setup not yet complete — needs config.js with Prowlarr Torznab URLs and qBittorrent connection. Service already in docker-compose.yml behind cross-seed profile. Key config notes: qBittorrent connects via vpn:8080 not qbittorrent:8080, Prowlarr URLs use prowlarr:9696 internally.
- Bazarr: configured with OpenSubtitles provider, English language profile ✓
- Seerr: setup wizard complete, connected to Jellyfin, Sonarr, Sonarr-anime, Radarr, Radarr-anime ✓
- Subdomain routing: all active services migrated from path-prefix to subdomain routing ✓
- Wildcard TLS cert: configured in Traefik, covers *.geo-front.net ✓
- AdGuard DNS rewrite: updated to wildcard *.geo-front.net → <server-lan-ip> ✓
- Prowlarr: indexers added, connected to all four arr instances ✓
- Sonarr/Radarr/Sonarr-anime/Radarr-anime: root folders configured, quality profiles synced
  via Recyclarr, connected to Prowlarr and qBittorrent ✓
- Recyclarr: being replaced by Profilarr — do not use. Media naming was configured and is still active in arr instances.
- Flaresolverr: replaced with Byparr (ghcr.io/thephaseless/byparr:latest), running on port 8191 ✓
- Watchtower: configured with WATCHTOWER_SCHEDULE=0 0 3 * * * (3am daily), TZ, and WATCHTOWER_CLEANUP. Debug mode removed. Automated image updates active.
- Byparr: removed broken compose healthcheck (curl not in image). Byparr has a built-in healthcheck at /health — no override needed.
- Watchtower scheduling and unattended-upgrades for apt: add to backlog as future automation items.
- Test download: verified end to end with The Lighthouse ✓
- Homepage: NGE theming - NERV terminal aesthetic, orange/black color scheme,
  themed service names (MAGI System title, Unit-01 for Jellyfin, etc)
- qBittorrent: password changed ✓

### DevOps
- Set up branch protection rules on master in GitHub
  (require PR, require compose validation to pass before merge)
- Create docs/dev-setup.md with full WSL dev machine setup steps
- Encode all manual server setup steps into Ansible playbook:
  - Samba install and share config
  - mergerfs + fstab setup
  - Tailscale install
  - Docker install
  - Directory structure creation (/mnt/data/media/movies, tv, anime, music, 
    /mnt/data/torrents)
  - seerr config directory pre-creation with correct ownership
- Test and validate full Ansible playbook end-to-end on a fresh Debian install

#### Done
- **CONFIGURE_SERVICE playbook**: created `docs/playbooks/CONFIGURE_SERVICE.md`, the companion to ADD_SERVICE for config-driven services whose value lives in a hand-authored config file containing secrets, external API calls, or destructive actions. Covers config classification (secret / environment-specific / safety-behavior), a mount-visibility-and-breadth precondition, a handoff checkpoint where the human fills secrets server-side, a one-time-vs-steady-state distinction, and a gated human-watched first run that verifies the intended END STATE, not just that execution completed. cross-seed is the worked example. ✓
- **Playbooks README / orchestrator**: created `docs/playbooks/README.md` as the index and router for the playbook set — routing guide (ADD vs REMOVE vs CONFIGURE), shared conventions stated once (SSH precondition, clean-tree, valid-config, the @claude two-round review cap, the root-owned-bind-mount chown chain, secrets-never-in-tracked-files), a capability map, and a roadmap of future playbooks. ✓
- **gitleaks secret scanning (CI)**: added `.github/workflows/gitleaks.yml` running gitleaks on every PR and on push to master, with custom rules in `.gitleaks.toml` for this repo's exposure surface (PT passkeys/announce URLs, *arr and Prowlarr API keys, RFC1918 addresses). Full git history audited (281 commits) — clean, no leaks. `GITLEAKS_VERSION` pinned and actions SHA-pinned since this is the supply-chain/secret-hygiene workflow itself. ✓

## Maintenance

- **VPN container scheduled restart**: Add a daily cron job on nerv to restart the vpn
  container at 4am to prevent AirVPN WireGuard session stalling.
  Command: `0 4 * * * docker restart vpn`
  Add via `crontab -e` as gendo on nerv. Also consider encoding in Ansible.

## Known Issues & Solutions

### Hardlinks not working with mergerfs across drives
mergerfs cannot create hardlinks across two different physical drives. Symptoms: `Cross-device link error` when running `ln` between `/mnt/data/torrents` and `/mnt/data/media`. Root cause: media was on disk1 and torrents on disk2. Fix applied May 20: moved media to disk2 so both torrents and media are on same physical drive. Hardlinks now work through mergerfs mount. Long term fix: reformat as LVM.

### Hardlink setup
For hardlinks to work, both the download client and arr apps must share the same filesystem. The TRaSH guide recommends qBittorrent mount as `/data/torrents` and arr apps as `/data`. This works only if torrents and media are on the same underlying physical drive. The upstream compose (`${DOWNLOAD_ROOT}:/data/torrents`) is correct per TRaSH — the issue was drive layout not volume mounts.

### Vaultwarden backup.env
- vaultwarden/docker-compose.yml references vaultwarden/backup.env for the rclone-backup service. This file must exist or docker compose config fails entirely, breaking variable resolution for ALL services (symptoms: labels missing from containers, env vars not interpolating). Fix: `touch ~/magi/vaultwarden/backup.env`. Add this to fresh install steps.

### Sonarr Anime series type
- Sonarr Anime series type must be set to Anime — there is no global default. Always add anime through Seerr where the default is configured, not directly in Sonarr Anime. If adding directly, manually set Series Type to Anime before saving or interactive search will use standard S01E01 format and return no results from Nyaa.

### Jellyfin series metadata conflicts
- When a long-running series has been rebooted or continued under a different arc, AniDB and TVDB may handle the season and episode structure differently, causing wrong images and episode metadata. Fix: go to the series in Jellyfin, use Identify to manually match to the correct TheTVDB entry, then go to Edit and lock all metadata fields to prevent AniDB from overwriting on future refreshes. Long term fix: split conflicting arcs into separate series in Sonarr Anime pointing to separate folders.

### Nyaa indexer settings (Prowlarr)
- Anime TV Nyaa indexer: enable "Improve Sonarr compatibility" and "Remove first season keywords"
- Anime Movies Nyaa indexer: enable "Improve Radarr compatibility"

### Quality Profile Sync
- **Recyclarr v8 custom format scoring broken**: Recyclarr v8 removed include templates.
  The guide-backed profile approach using trash_id does not auto-sync release group tiers
  (HD Bluray Tier 01/02/03, WEB Tier 01/02/03). Custom format scores for these groups
  do not apply correctly resulting in +0 scores for many releases in interactive search.
  Solution: Replace Recyclarr with Profilarr + Dumpstarr database.

### Vaultwarden
- **Vaultwarden env file**: after creating `~/magi/vaultwarden/.env`, must use
  `docker compose down vaultwarden && docker compose up -d vaultwarden`
  (not just restart) for env vars to be picked up by the container

### Permissions
- **seerr runs as node:node (UID 1000)** not as the configured USER_ID. 
  Pre-create config dir before stack starts:
  `sudo mkdir -p ~/magi/seerr/logs && sudo chown -R 1000:1000 ~/magi/seerr`
  This is encoded in Ansible but must be done manually on fresh installs until
  Ansible is fully tested.
- **update-config.sh requires ownership match** between running user and config dirs.
  Containers run as USER_ID (currently gendo UID 1000). Run script as gendo or with
  sudo if ownership mismatch occurs.
- **gendo UID changed from 1001 to 1000** during setup to avoid constant permission
  conflicts. shinji is now UID 1002. Containers run as UID 1000 (gendo).

### Networking
- **qBittorrent Server domains**: must be set to `*` (wildcard), not the subdomain hostname.
  The arr apps connect internally via vpn:8080 and will fail if Server domains is restricted.
- **AirVPN WireGuard session stalling**: VPN may appear connected but traffic stops
  after extended uptime. Workaround: restart vpn container manually with
  `docker restart vpn`. Long term fix: daily cron restart at 4am (see Maintenance).
- **AdGuard port 68 conflicts with host dhcpcd** — removed ports 68/tcp and 68/udp
  from adguardhome/docker-compose.yml. AdGuard DHCP server disabled, DNS only.
- **Verizon FiOS router DNS locked down** — cannot set custom DNS at router level.
  Workaround: set DNS manually per device to server LAN IP (192.168.x.x).
  Long term fix: put own router behind Verizon box in IP passthrough mode.
- **mergerfs mount drops if user sessions terminated abruptly**:
  `sudo fusermount -uz /mnt/data && sudo umount -f /mnt/data && sudo mount /mnt/data`
- **COMPOSE_FILE must explicitly include docker-compose.override.yml** — Docker does
  not auto-load the override file when COMPOSE_FILE is set in .env. Always include:
  `COMPOSE_FILE=docker-compose.yml:adguardhome/docker-compose.yml:docker-compose.override.yml`

### Healthchecks
- Always use wget not curl. Curl is frequently absent from minimal images. Always include timeout and start_period. If the image has a built-in healthcheck (e.g. byparr), remove the compose override entirely rather than duplicating it.

### Watchtower debug mode overrides schedule
- WATCHTOWER_DEBUG=true causes watchtower to poll continuously and ignore WATCHTOWER_SCHEDULE. Always remove debug mode in production.

### GitHub Claude bot polling
- The bot edits its comment in place as it works. Do not poll based on comment count. Poll based on comment body containing "Claude finished" which signals the review is complete.

### Git / GitHub
- **Fork always defaults to upstream base on PR creation** — always use gh CLI:
  `gh pr create --base master --head your-branch --title "title"`
- **GitHub Actions comment-triggered workflows only run from master** — workflows
  on feature branches won't trigger on PR comments until merged to master.
### Server Setup
- **vaultwarden/backup.env must exist**: touch ~/magi/vaultwarden/backup.env before bringing stack up, or compose variable resolution breaks globally.
- **sudo not installed on fresh Debian** — must su to root first and install:
  `su - && apt install curl sudo git htop ufw -y && usermod -aG sudo gendo`
- **Node.js via apt is outdated** — always install via nvm:
  `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash`
  then `nvm install --lts`
- **SSH into server drops when terminating own session** — use tmpuser or shinji
  when needing to change gendo's UID/session while logged in as gendo.
- **Tailscale blocks local IP SSH after ufw enabled** — this is intentional.
  Only connect via Tailscale IP (100.x.x.x) after setup.

### Vaultwarden-backup crash loop
The rclone-backup container will crash loop at startup if backup.env exists but rclone config is missing or empty. Crash loop at 9000+ restarts was causing Docker daemon instability affecting other containers. Fix: configure rclone properly with a valid remote before starting the container. The rclone config must be at ~/magi/vaultwarden/backup/rclone/rclone.conf and backup.env must have RCLONE_REMOTE_NAME, RCLONE_REMOTE_DIR, CRON, and TIMEZONE at minimum.

### OpenSubtitles auth error in Bazarr
opensubtitles.org and opensubtitles.com are completely separate services with separate accounts. Bazarr's provider is opensubtitles.com — must have a .com account specifically. Using .org credentials causes AuthenticationError and 12-hour throttle.

### Containers stopping unexpectedly
If multiple containers stop simultaneously check docker inspect on vaultwarden-backup for restart count. A crash-looping container at high restart counts taxes the Docker daemon. Stop it with docker stop and fix the root cause before restarting.

### Private tracker ratio protection
Auto-upgrades in Radarr/Sonarr can trigger new downloads that affect ratio on private trackers. Set Upgrade Until cutoff in quality profiles to limit automatic upgrade behavior.

### Cross-seed data-based recovery (reseed existing library after unsatisfieds)
When torrents are removed from the client but the data still exists on disk, cross-seed can rematch and reseed without re-downloading. Key setup: mount the FULL data root into the cross-seed container (not just the torrents dir) so it can see the media library; point `dataDirs` at the media paths; use `matchMode: "flexible"` for renamed library files; keep `skipRecheck: false` so injected torrents are verified before seeding; `linkDir` must be on the same mount as the data (cross-device hardlinks fail). Run as a one-shot search (`docker compose run --rm cross-seed search`), not daemon, for a recovery pass. NOTE: injected torrents can land in a stopped/"done" state and silently not announce to the tracker — force-resume them and confirm they appear as seeding on the tracker. Data-based matching only recovers content still on disk; content that was downloaded-then-deleted is unrecoverable and is a tracker-waiver conversation, not a technical fix.

### "Indexer unavailable" in the arrs usually means a failed GRAB, not a down indexer
Search is a cheap call that succeeds while the actual torrent-file fetch through Prowlarr fails (e.g. tracker returning an error page instead of a `.torrent`, or a download restriction). Diagnose by checking the Prowlarr-side download error, not just the arr's indexer status.

### PT unsatisfied-torrents incident (resolved)
Root cause was a per-indexer seed-time value that counted download time toward the seed window, so torrents fell short of the 10-day minimum; a downstream removal then deleted them. Fixed by removing the bad timer (everything now seeds indefinitely), recovering all on-disk content via cross-seed, and requesting a staff waiver for the genuinely-unrecoverable remainder (deleted data / bulk-removed torrents). Set PT per-indexer seed time correctly going forward.

## Dev Machine Setup (WSL Ubuntu 24.04)

### One-time setup
```bash
# GitHub CLI
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install gh -y

# Authenticate GitHub CLI
gh auth login
# Choose: GitHub.com, HTTPS, authenticate via browser
# Copy URL shown and open in Windows browser manually (WSL can't open browser)

# Set default repo
gh repo set-default giovanndiv/magi

# Node.js via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts

# Claude Code
npm install -g @anthropic-ai/claude-code

# Git identity (use real GitHub email, themed name is fine)
git config --global user.email "your@github.email"
git config --global user.name "gendo"

# SSH key for GitHub (do inside WSL, not Windows)
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
# Add to GitHub Settings -> SSH keys
ssh -T git@github.com  # verify

# Clone repo
cd ~
git clone git@github.com:giovanndiv/magi.git
cd magi
git remote add upstream https://github.com/AdrienPoupa/docker-compose-nas.git
gh repo set-default giovanndiv/magi
```

### Git workflow
```bash
# Always branch before making changes
git checkout -b feat/your-feature

# Commit and push
git add .
git commit -m "descriptive message"
git push origin feat/your-feature

# Open PR (always use gh CLI to avoid upstream base issue)
gh pr create --base master --head feat/your-feature --title "your title"

# After merge, clean up
git checkout master
git pull origin master
git branch -d feat/your-feature
git push origin --delete feat/your-feature
```

### VS Code setup
- Install WSL extension (Microsoft)
- Connect to WSL: Ctrl+Shift+P -> "WSL: Connect to WSL"
- Open repo: `code ~/magi` from WSL terminal
- Claude Code panel: click Spark icon in sidebar
- Run `/init` in Claude Code on first open in a new repo

## Architecture Reference

### Naming Conventions
- Machine hostname: `nerv`
- Repo/stack: `magi`  
- Standard user: `shinji` (UID 1002)
- Admin user: `gendo` (UID 1000) — owns Docker, containers, config files
- Domain: `geo-front.net` (Cloudflare)
- Tailscale network: AT-Field
- Docker network: `nerv`

### Key Paths
- Repo: `/home/gendo/magi`
- Media: `/mnt/data/media/{movies,tv,anime,music}`
- Downloads: `/mnt/data/torrents/{movies,tv,anime}`
- Ratio building downloads: `/mnt/data/torrents/ratio-building`
- Drives: `/mnt/disk1` (sda), `/mnt/disk2` (sdb), merged at `/mnt/data`
- Samba share: `\\<tailscale-ip>\geofront` → `/mnt/data`

### Container UID Notes
- Most containers run as UID 1000 (gendo) via USER_ID=1000 in .env
- Exception: seerr runs as node:node (also UID 1000 in most Node images)
- Config dirs must be owned by UID 1000 for containers to write to them
- run `sudo chown -R 1000:1000 ~/magi/<service>` if permission errors occur
