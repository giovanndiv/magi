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

Key .env values to set:
- `USER_ID=1000`
- `GROUP_ID=1000`
- `TIMEZONE=America/New_York`
- `CONFIG_ROOT=.`
- `DATA_ROOT=/mnt/data`
- `DOWNLOAD_ROOT=/mnt/data/torrents`
- `HOSTNAME=magi.geo-front.net`
- `BASE_HOSTNAME=geo-front.net`
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
sudo chown -R 1000:1000 /mnt/data/
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
- Navigate to `https://magi.geo-front.net/qbittorrent`
- Tools > Options > Web UI: change default password
- Tools > Options > Downloads: set default save path to `/data/torrents`
- Tools > Options > Web UI: enable subnet whitelist for Docker subnet (172.18.0.0/16), enable localhost bypass
- Tools > Options > Connection: set listening port to 7123 (AirVPN forwarded port)
- Update `QBITTORRENT_PASSWORD` in .env, restart homepage

### 3.4 Prowlarr
- Navigate to `https://magi.geo-front.net/prowlarr`
- Settings > Indexers: Add FlareSolverr proxy (host: `http://flaresolverr:8191`, tag: `flaresolverr`)
- Settings > Apps: Add Sonarr, Sonarr-anime, Radarr, Radarr-anime (Full Sync)
  - Sonarr: `http://sonarr:8989`, API key from .env, no tags
  - Sonarr-anime: `http://sonarr-anime:8990`, API key from .env, tag: `anime-tv`
  - Radarr: `http://radarr:7878`, API key from .env, no tags
  - Radarr-anime: `http://radarr-anime:7979`, API key from .env, tag: `anime-movie`
- Add indexers: 1337x, YTS (no flaresolverr tag), Nyaa TV (tag: `anime-tv`), Nyaa Movies (tag: `anime-movie`)

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

### 3.6 Recyclarr
```bash
# Fix permissions
sudo chown -R 1000:1000 ~/magi/recyclarr/

# Create config
docker exec -it recyclarr recyclarr config create

# Create configs directory and add recyclarr.yml with all four instances
# See recyclarr/configs/recyclarr.yml on server for current config
# Create recyclarr/secrets.yml with API keys (never commit)

docker exec -it recyclarr recyclarr sync
```

### 3.7 Jellyfin
- Navigate to `https://magi.geo-front.net/jellyfin`
- Complete setup wizard (create admin account)
- Dashboard > Playback > Transcoding: set Hardware Acceleration to Intel QuickSync/VAAPI, device `/dev/dri/renderD128`
- Add libraries: Movies (`/data/media/movies`), TV (`/data/media/tv`), Anime (`/data/media/anime`), Music (`/data/media/music`)
- Dashboard > API Keys: create key, add to .env as `JELLYFIN_API_KEY`

### 3.8 Bazarr
- Navigate to `https://magi.geo-front.net/bazarr`
- Settings > Sonarr: enable, host=`sonarr`, port=`8989`, API key from .env
- Settings > Radarr: enable, host=`radarr`, port=`7878`, API key from .env
- Settings > Languages: create English profile with cutoff=English
- Settings > Languages: enable default profile for Series and Movies
- Settings > Providers: add OpenSubtitles.com with credentials
- Settings > Subtitles: enable Automatic Subtitles Synchronization

### 3.9 Seerr
- Navigate to `https://magi.geo-front.net/seerr`
- Complete setup wizard: connect to Jellyfin, then Sonarr/Radarr instances
- Settings > General: copy API key, add to .env as `SEERR_API_KEY`

### 3.10 Daily VPN Restart Cron
```bash
crontab -e
# Add: 0 4 * * * docker restart vpn
```

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
