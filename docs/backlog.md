# MAGI Project Backlog

## Pending Tasks

### Infrastructure
- AUTOBRR_HOSTNAME: already added to .env.example ✓
- VAULTWARDEN_HOSTNAME: already added to .env.example ✓
- Rename docker network from `docker-compose-nas` to `nerv` in docker-compose.yml
  and adguardhome/docker-compose.yml ✓
- Add recyclarr to docker-compose.yml (image: ghcr.io/recyclarr/recyclarr:8, 
  user 1000:1000, RECYCLARR_CREATE_CONFIG=true)
- Daily VPN restart cron: added via crontab -e as gendo (0 4 * * * docker restart vpn) ✓
- Set up snapraid alongside mergerfs for parity protection on the 2x14TB drives
- Get a second SSD for container config storage (current 128GB M.2 is OS + configs)
- Consider upgrading RAM from 16GB if running LLM (Ollama) in future

### Services To Add
- **Cross-seed**: deferred until library grows. Already in docker-compose.yml behind profile. Enable with
  `COMPOSE_PROFILES=cross-seed`. Automates cross-seeding torrents across trackers for better
  availability. Needs configuration at `~/magi/cross-seed/config.js`.
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

### Configuration
- Vaultwarden backup: configure rclone-backup service with a cloud destination (S3, Google Drive, etc.) and populate ~/magi/vaultwarden/backup.env with rclone credentials. Deferred until Vaultwarden is in active use.
- Sonarr Anime: when adding series, always set Series Type to Anime (not Standard). Also set default Series Type to Anime in Seerr → Settings → Sonarr Anime instance.
- Jellyfin: configure AniDB/AniList metadata plugins when anime content added
- Jellyfin: verify QuickSync QSV is actually being used during transcoding 
  (play content and check Dashboard > Activity)
- Bazarr: configured with OpenSubtitles provider, English language profile ✓
- Seerr: setup wizard complete, connected to Jellyfin, Sonarr, Sonarr-anime, Radarr, Radarr-anime ✓
- Subdomain routing: all active services migrated from path-prefix to subdomain routing ✓
- Wildcard TLS cert: configured in Traefik, covers *.geo-front.net ✓
- AdGuard DNS rewrite: updated to wildcard *.geo-front.net → 192.168.1.201 ✓
- Prowlarr: indexers added, connected to all four arr instances ✓
- Sonarr/Radarr/Sonarr-anime/Radarr-anime: root folders configured, quality profiles synced
  via Recyclarr, connected to Prowlarr and qBittorrent ✓
- Recyclarr: being replaced by Profilarr — do not use. Media naming was configured and is still active in arr instances.
- Flaresolverr: replaced with Byparr (ghcr.io/thephaseless/byparr:latest), running on port 8191 ✓
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

## Maintenance

- **VPN container scheduled restart**: Add a daily cron job on nerv to restart the vpn
  container at 4am to prevent AirVPN WireGuard session stalling.
  Command: `0 4 * * * docker restart vpn`
  Add via `crontab -e` as gendo on nerv. Also consider encoding in Ansible.

## Known Issues & Solutions

### Vaultwarden backup.env
- vaultwarden/docker-compose.yml references vaultwarden/backup.env for the rclone-backup service. This file must exist or docker compose config fails entirely, breaking variable resolution for ALL services (symptoms: labels missing from containers, env vars not interpolating). Fix: `touch ~/magi/vaultwarden/backup.env`. Add this to fresh install steps.

### Sonarr Anime series type
- When adding anime series, Series Type must be set to Anime (not Standard). Standard uses S01E01 format which Nyaa doesn't index — Anime type uses absolute episode numbering. Set default in Seerr → Settings → Sonarr Anime to avoid fixing this per-series.

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
