# MAGI Setup Runbook

## Phase 1: Server Bootstrap (Manual, One-Time)

### 1.1 Debian Install
- Install Debian Bookworm headless on target machine
- During install: create user gendo, set root password
- Boot into fresh system

### 1.2 Initial System Config (as root)
```bash
su -
apt install curl sudo git htop ufw ca-certificates -y
usermod -aG sudo gendo
usermod -aG docker gendo  # after docker install
timedatectl set-timezone America/New_York
exit
```

### 1.3 Tailscale Install
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# Authenticate via URL shown
tailscale ip  # note the 100.x.x.x IP
```

### 1.4 SSH Hardening (lock to Tailscale only)
```bash
sudo nano /etc/ssh/sshd_config
# Add: ListenAddress <tailscale-ip>
sudo systemctl restart sshd
# Verify SSH works via Tailscale IP before closing current session
sudo ufw allow in on tailscale0 to any port 22
sudo ufw enable
```

### 1.5 Drive Setup
```bash
# Format drives if needed
sudo mkfs.ext4 /dev/sda
sudo mkfs.ext4 /dev/sdb

# Get UUIDs
sudo blkid /dev/sda
sudo blkid /dev/sdb

# Create mount points
sudo mkdir -p /mnt/disk1 /mnt/disk2 /mnt/data

# Install mergerfs
sudo apt install mergerfs -y

# Add to /etc/fstab:
# UUID=<sda-uuid> /mnt/disk1 ext4 defaults 0 2
# UUID=<sdb-uuid> /mnt/disk2 ext4 defaults 0 2
# /mnt/disk1:/mnt/disk2 /mnt/data fuse.mergerfs defaults,allow_other,use_ino,cache.files=off,moveonenospc=true,dropcacheonclose=true,minfreespace=250G 0 0

sudo mount -a
```

### 1.6 Docker Install
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker gendo
# Log out and back in for group change to take effect
```

### 1.7 Samba Install
```bash
sudo apt install samba -y
# Add to /etc/samba/smb.conf:
# [geofront]
#    path = /mnt/data
#    browseable = yes
#    writeable = yes
#    valid users = gendo
sudo smbpasswd -a gendo  # set password interactively
sudo systemctl restart smbd
sudo ufw allow in on tailscale0 to any port 445
sudo ufw allow in on tailscale0 to any port 139
```

## Phase 2: Stack Bootstrap

### 2.1 Clone Repo and Configure .env
```bash
cd ~
git clone git@github.com:giovanndiv/magi.git
cd magi
cp .env.example .env
nano .env
```

Routing is subdomain-based — each service gets its own subdomain (e.g. `sonarr.geo-front.net`).
All service hostname variables are derived from `BASE_HOSTNAME`.

Key .env values to set:
- `USER_ID=1000`
- `GROUP_ID=1000`
- `TIMEZONE=America/New_York`
- `CONFIG_ROOT=.`
- `DATA_ROOT=/mnt/data`
- `DOWNLOAD_ROOT=/mnt/data/torrents`
- `HOSTNAME=magi.geo-front.net`
- `BASE_HOSTNAME=geo-front.net`
- `SONARR_HOSTNAME=sonarr.${BASE_HOSTNAME}`
- `SONARR_ANIME_HOSTNAME=sonarr-anime.${BASE_HOSTNAME}`
- `RADARR_HOSTNAME=radarr.${BASE_HOSTNAME}`
- `RADARR_ANIME_HOSTNAME=radarr-anime.${BASE_HOSTNAME}`
- `BAZARR_HOSTNAME=bazarr.${BASE_HOSTNAME}`
- `LIDARR_HOSTNAME=lidarr.${BASE_HOSTNAME}`
- `PROWLARR_HOSTNAME=prowlarr.${BASE_HOSTNAME}`
- `QBITTORRENT_HOSTNAME=qbittorrent.${BASE_HOSTNAME}`
- `JELLYFIN_HOSTNAME=jellyfin.${BASE_HOSTNAME}`
- `CLEANUPARR_HOSTNAME=cleanuparr.${BASE_HOSTNAME}`
- `CALIBRE_HOSTNAME=calibre.${BASE_HOSTNAME}`
- `SUGGESTARR_HOSTNAME=suggestarr.${BASE_HOSTNAME}`
- `AUTOBRR_HOSTNAME=autobrr.${BASE_HOSTNAME}`
- `VAULTWARDEN_HOSTNAME=vaultwarden.${BASE_HOSTNAME}`
- `LETS_ENCRYPT_EMAIL=your@email.com`
- `COMPOSE_FILE=docker-compose.yml:adguardhome/docker-compose.yml:docker-compose.override.yml`
- `COMPOSE_PROFILES=adguardhome,flaresolverr`
- `AIRVPN_PRIVATE_KEY=` (from AirVPN config generator)
- `AIRVPN_PRESHARED_KEY=` (from AirVPN config generator)
- `AIRVPN_ADDRESSES=` (IPv4 only from AirVPN config, e.g. 10.x.x.x/32)
- `AIRVPN_COUNTRY=United States`
- `AIRVPN_PORT=7123` (forwarded port from AirVPN client area)

### 2.2 Create Required Directories
```bash
sudo mkdir -p /mnt/data/media/movies
sudo mkdir -p /mnt/data/media/tv
sudo mkdir -p /mnt/data/media/anime
sudo mkdir -p /mnt/data/media/music
sudo mkdir -p /mnt/data/torrents/tv-sonarr
sudo mkdir -p /mnt/data/torrents/tv-sonarr-anime
sudo mkdir -p /mnt/data/torrents/radarr
sudo mkdir -p /mnt/data/torrents/radarr-anime
sudo mkdir -p /mnt/data/torrents/ratio-building
sudo chown -R 1000:1000 /mnt/data/
sudo chown 1000:1000 /mnt/data/torrents/ratio-building
```

### 2.3 Pre-create seerr logs directory
```bash
mkdir -p ~/magi/seerr/logs
chown -R 1000:1000 ~/magi/seerr/
```

### 2.4 Bring Stack Up
```bash
cd ~/magi
docker compose up -d
docker compose ps  # verify all healthy
```

## Phase 3: Service Configuration (Manual, Post-Stack)

### 3.1 AdGuard Home
- Navigate to `http://<tailscale-ip>:3000`
- Complete setup wizard (set credentials, listen interfaces)
- Add DNS blocklists: AdGuard DNS filter, EasyList, EasyPrivacy
- Set DNS manually on each device to server LAN IP

### 3.2 Populate API Keys
```bash
cd ~/magi
./update-config.sh
grep "_API_KEY" .env  # verify keys populated
```
- Manually add to .env: `JELLYFIN_API_KEY` (from Jellyfin Dashboard > API Keys)
- Manually add to .env: `SEERR_API_KEY` (from Seerr Settings > General)

### 3.3 qBittorrent
- Navigate to `https://qbittorrent.geo-front.net`
- Tools > Options > Web UI: change default password
- Tools > Options > Downloads: set default save path to `/data/torrents`
- Tools > Options > Web UI: enable subnet whitelist for Docker subnet (172.18.0.0/16), enable localhost bypass
- Tools > Options > Connection: set listening port to 7123 (AirVPN forwarded port)
- Update `QBITTORRENT_PASSWORD` in .env, restart homepage

### 3.4 Prowlarr
- Navigate to `https://prowlarr.geo-front.net`
- Settings > Indexers: Add FlareSolverr proxy (host: `http://flaresolverr:8191`, tag: `flaresolverr`)
- Settings > Apps: Add Sonarr, Sonarr-anime, Radarr, Radarr-anime (Full Sync)
  - Sonarr: `http://sonarr:8989`, API key from .env, no tags
  - Sonarr-anime: `http://sonarr-anime:8990`, API key from .env, tag: `anime-tv`
  - Radarr: `http://radarr:7878`, API key from .env, no tags
  - Radarr-anime: `http://radarr-anime:7979`, API key from .env, tag: `anime-movie`
- Add indexers as needed (tag with `anime-tv` for Sonarr-anime routing, `anime-movie` for Radarr-anime routing)

### 3.5 Sonarr / Sonarr-anime / Radarr / Radarr-anime
For each instance:
- Settings > Download Clients > Add qBittorrent:
  - Host: `vpn`, Port: `8080`
  - Username: `admin`, Password: (qbit password)
  - Category: `tv-sonarr` / `tv-sonarr-anime` / `radarr` / `radarr-anime`
- Settings > Media Management > Root Folders:
  - Sonarr: `/data/media/tv`
  - Sonarr-anime: `/data/media/anime`
  - Radarr: `/data/media/movies`
  - Radarr-anime: `/data/media/movies`

### 3.6 Quality Profile Sync (Profilarr)
NOTE: Recyclarr has been replaced by Profilarr. Do not set up Recyclarr on new installs.

Profilarr uses the Dumpstarr database and has a web UI at https://profilarr.geo-front.net.
See backlog for full Profilarr setup steps.

Media naming is handled directly in each arr instance Settings → Media Management:
- Sonarr/Sonarr-anime: jellyfin-tvdb series folder format
- Radarr: jellyfin-tmdb movie folder format
- Radarr-anime: jellyfin-anime-tmdb movie folder format

### 3.7 Jellyfin
- Navigate to `https://jellyfin.geo-front.net`
- Complete setup wizard (create admin account)
- Dashboard > Playback > Transcoding: set Hardware Acceleration to Intel QuickSync/VAAPI, device `/dev/dri/renderD128`
- Add libraries: Movies (`/data/media/movies`), TV (`/data/media/tv`), Anime (`/data/media/anime`), Music (`/data/media/music`)
- Dashboard > API Keys: create key, add to .env as `JELLYFIN_API_KEY`

### 3.8 Bazarr
- Navigate to `https://bazarr.geo-front.net`
- Settings > Sonarr: enable, host=`sonarr`, port=`8989`, API key from .env
- Settings > Radarr: enable, host=`radarr`, port=`7878`, API key from .env
- Settings > Languages: create English profile with cutoff=English
- Settings > Languages: enable default profile for Series and Movies
- Settings > Providers: add OpenSubtitles.com with credentials
- Settings > Subtitles: enable Automatic Subtitles Synchronization

### 3.9 Daily VPN Restart Cron
```bash
crontab -e
# Add: 0 4 * * * docker restart vpn
```

### 3.10 Seerr
- Navigate to `https://seerr.geo-front.net`
- Complete setup wizard:
  - Connect Jellyfin using internal URL: `http://jellyfin`, port `8096`, no SSL
  - Scan libraries
  - Add Radarr: hostname=`radarr`, port=`7878`, quality profile=`HD Bluray + WEB`,
    root folder=`/data/media/movies`, external URL=`https://radarr.geo-front.net`
  - Add Radarr Anime: hostname=`radarr-anime`, port=`7979`, quality profile=`[Anime] Remux-1080p`,
    root folder=`/data/media/movies`, external URL=`https://radarr-anime.geo-front.net`
  - Add Sonarr: hostname=`sonarr`, port=`8989`, quality profile=`WEB-1080p`,
    root folder=`/data/media/tv`, season folders=enabled,
    external URL=`https://sonarr.geo-front.net`
  - Add Sonarr Anime: hostname=`sonarr-anime`, port=`8990`, quality profile=`[Anime] Remux-1080p`,
    root folder=`/data/media/anime`, series type=Anime, season folders=enabled,
    external URL=`https://sonarr-anime.geo-front.net`
- Settings > General: copy API key, add to .env as `SEERR_API_KEY`, restart homepage
- Settings > General: set Application URL to `https://seerr.geo-front.net`

### 3.11 AdGuard DNS Wildcard Rewrite
- Navigate to AdGuard → Filters → DNS Rewrites
- Add rewrite: `*.geo-front.net` → `192.168.1.201` (server LAN IP)
- This ensures all service subdomains resolve to LAN IP for local clients

### 3.12 Vaultwarden
- Pre-create backup.env: `touch ~/magi/vaultwarden/backup.env`
  (Required — without this file, docker compose config fails and breaks variable resolution for all services)
- Add `vaultwarden/docker-compose.yml` to `COMPOSE_FILE` in `.env`
- Add `VAULTWARDEN_HOSTNAME=vaultwarden.geo-front.net` to `.env`
- Create `~/magi/vaultwarden/.env` with:
  ```
  SIGNUPS_ALLOWED=false
  ADMIN_TOKEN=$(openssl rand -base64 48)
  ```
- Create account at `https://vaultwarden.geo-front.net` BEFORE adding `SIGNUPS_ALLOWED=false`
- Save ADMIN_TOKEN as secure note in Vaultwarden
- Access admin panel at `https://vaultwarden.geo-front.net/admin`
- Install Bitwarden browser extension, set server URL to `https://vaultwarden.geo-front.net`
- IMPORTANT: Use `docker compose down`/`up` (not restart) for env file changes to take effect

**Vaultwarden backup setup:**
- Create rclone config directory: `mkdir -p ~/magi/vaultwarden/backup/rclone`
- Fix ownership: `sudo chown -R 1000:1000 ~/magi/vaultwarden/backup`
- Configure rclone on local machine, copy rclone.conf to `~/magi/vaultwarden/backup/rclone/rclone.conf` on server
- Remote name must be `RcloneBackup`
- Populate `~/magi/vaultwarden/backup.env` with `RCLONE_REMOTE_NAME`, `RCLONE_REMOTE_DIR`, `CRON`, `ZIP_PASSWORD`, `BACKUP_KEEP_DAYS`, `TIMEZONE`
- Start container: `docker compose up -d vaultwarden-backup`
- Test: `docker exec vaultwarden-backup /app/backup.sh`

### 3.13 Autobrr
- Add `AUTOBRR_HOSTNAME=autobrr.geo-front.net` to `.env`
- `mkdir -p /mnt/data/torrents/ratio-building && chown 1000:1000 /mnt/data/torrents/ratio-building`
- `docker compose up -d autobrr`
- Navigate to `https://autobrr.geo-front.net` and create admin account
- Settings → API Keys: copy key, add to `.env` as `AUTOBRR_API_KEY`
- `docker compose restart homepage`
- Settings → Clients: add qBittorrent (host=`vpn`, port=`8080`, username=`admin`, password=qbit password)
- Settings → Clients: add Sonarr (host=`sonarr`, port=`8989`, no SSL, API key from `.env`)
- Settings → Clients: add Radarr (host=`radarr`, port=`7878`, no SSL, API key from `.env`)
- Settings → Clients: add Sonarr-anime (host=`sonarr-anime`, port=`8990`, no SSL, API key from `.env`)
- Settings → Clients: add Radarr-anime (host=`radarr-anime`, port=`7979`, no SSL, API key from `.env`)
- Settings → IRC: configure IRC networks as needed for your indexers
- Settings → Indexers: add indexers as needed
- Filters → Create filters to route grabs to the appropriate arr instance or qBittorrent category

## Phase 4: Verification

### 4.1 VPN Leak Check
```bash
docker exec vpn wget -q6 -O- ifconfig.me  # should fail or return AirVPN IP
docker exec qbittorrent curl -s ifconfig.me  # should return AirVPN IP
```

### 4.2 Test Download
- Add a test torrent via Sonarr or Radarr
- Verify it appears in qBittorrent
- Verify it imports to media folder after completion
- Verify Jellyfin picks it up

### 4.3 QuickSync Verification
- Play content in Jellyfin that requires transcoding
- Dashboard > Activity: confirm hardware transcoder is active

