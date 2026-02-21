# qBittorrent Install and Docker-Compose Guide

qBittorrent is a free and open-source BitTorrent client. It serves as a download client for Sonarr and Radarr, automatically downloading torrent files and managing the torrent queue for your media library. It provides efficient torrent management, seeding control, and integration with your media server stack.

## Directory Structure

qBittorrent uses the following directory structure (defined in the main media README):

```
/docker/servarr/qbittorrent
├── /config                 # qBittorrent configuration, settings, and database

/data/downloads/qbittorrent
├── /completed              # Completed torrent downloads (ready for post-processing)
└── /incomplete             # In-progress torrent downloads (being downloaded)
```

### Create qBittorrent Directories

Before starting qBittorrent, create the required directories:

```bash

# Create qBittorrent download directories
mkdir -p /data/downloads/qbittorrent/completed
mkdir -p /data/downloads/qbittorrent/incomplete

# Set proper permissions (adjust 1000:1000 to your PGID:PUID)
chown -R 1000:1000 /docker/servarr/qbittorrent
chown -R 1000:1000 /data/downloads/qbittorrent

```

Or use this one-liner:
```bash
mkdir -p /docker/servarr/qbittorrent /data/downloads/qbittorrent/{completed,incomplete} && chown -R 1000:1000 /docker/servarr/qbittorrent /data/downloads/qbittorrent
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add qBittorrent service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host. This snippet intentionally removes any personal or machine-specific absolute paths.

```yaml
qbittorrent:
  cpu_shares: 90
  command: []
  container_name: qbittorrent
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"  # replace with your timezone
    WEBUI_PORT: "8080"
  hostname: qbittorrent
  image: lscr.io/linuxserver/qbittorrent:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 8080
      published: "8080"
      protocol: tcp
    - mode: ingress
      target: 6881
      published: "6881"
      protocol: tcp
    - mode: ingress
      target: 6881
      published: "6881"
      protocol: udp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/qbittorrent  # replace with your config path
      target: /config
      bind:
        create_host_path: true
    - type: bind
      source: /data/downloads/qbittorrent/completed  # Completed downloads directory
      target: /data/torrents/completed
      bind:
        create_host_path: true
    - type: bind
      source: /data/downloads/qbittorrent/incomplete  # In-progress downloads directory
      target: /data/torrents/incomplete
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/qbittorrent` with your config storage path (typically `/docker/servarr/qbittorrent`).
- `/data/downloads/qbittorrent/completed` - where finished torrent downloads are stored (Sonarr/Radarr collect from here)
- `/data/downloads/qbittorrent/incomplete` - where active downloads are stored temporarily
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- Port `8080` is the web UI port (configured via `WEBUI_PORT` environment variable).
- Ports `6881` (TCP/UDP) are used for DHT, LSD, and PEX peer discovery (optional; can be changed).
- The config volume stores qBittorrent's settings, configuration, database, and download history.


## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d qbittorrent
```

## 4. Verify

- Check logs: `docker compose logs -f qbittorrent`
- Visit `http://<host-ip>:8080` to access the qBittorrent web UI (replace `<host-ip>` with the server IP).
- Default credentials: username `admin`, password `adminPassword` (change these immediately).

## 5. Initial Setup

When you first access the qBittorrent web UI, perform these essential configuration steps:

1. Open `http://<host-ip>:8080` in your browser
2. Log in with default credentials (username: `admin`, password: `adminPassword`)
3. Navigate to `Tools` -> `Options` -> `Web UI` and change the password immediately for security
4. Navigate to `Tools` -> `Options` and configure the following critical settings:

### Connection Settings:
1. Go to `Tools` -> `Options` -> `Connection`
   - **Listening Port**: Set to `6881` (or another high random port if preferred; must match compose.yaml)
   - **UPnP/NAT-PMP Port Mapping**: Disabled (if behind proxy/VPN)
   - **Proxy Server**: Configure if using VPN or proxy service
   - **Type**: Select SOCKS5 if using a VPN proxy

### Download Settings:
1. Go to `Tools` -> `Options` -> `Downloading`
   - **Default Save Path**: Set to `/data/torrents/completed` (matches your container volume mount)
   - **Keep Incomplete Torrents in**: `/data/torrents/incomplete` (optional but recommended)
   - **Use Completed Folder**: Checked
   - **Pre-Allocate All Files**: Enabled (uses more disk I/O but prevents fragmentation)
   - **Pause Them Upon Download**: Can be checked if you want manual torrent filtering

### Bittorrent Settings:
1. Go to `Tools` -> `Options` -> `Bittorrent`
   - **Seeding Limits**: Set time or ratio limits per your preference:
     - **Max Ratio**: `2.0` (recommended: 2x the download size, then stop seeding)
     - **Max Seeding Time**: `1440` minutes (1 day, optional)
   - **Torrent Queueing**: 
     - **Max Active Downloads**: `3` (adjust based on your bandwidth)
     - **Max Active Torrents**: `5`
   - **Share Ratio Limit**: `2.0`

### Advanced Options:
1. Go to `Tools` -> `Options` -> `Advanced`
   - **Enable UPnP**: Disabled (unless you specifically need it)
   - **Enable DHT**: Enabled (for better peer discovery)
   - **Enable PEX**: Enabled (for better peer discovery)
   - **Enable LSD**: Enabled (for local network peer discovery)

### Web UI Security:
1. Go to `Tools` -> `Options` -> `Web UI`
   - **Authentication**: Enable `Use authentication`
   - **Username**: `admin` (or change to your preference)
   - **Password**: Change from default to a strong password
   - **Use HTTPS**: Disabled (can be enabled if you have proper SSL certificates)

Click `Save` to apply all changes.

## 6. Connect qBittorrent to Sonarr and Radarr

qBittorrent acts as a download client for Sonarr and Radarr, automatically downloading torrents for requested content.

### Prerequisites:
- Sonarr and Radarr must be running and reachable from the qBittorrent host (same Docker network or host IP and port).
- You will need the qBittorrent username and password for authentication.
- Port `8080` (or your configured web UI port) must be accessible to Sonarr and Radarr.

### Change qBittorrent password:

1. Open qBittorrent web UI at `http://<host-ip>:8080`
2. Navigate to `Tools` -> `Options` -> `Web UI`
3. Change the password from the default `adminPassword` to a secure password
4. Click `Save` to apply changes

### Add qBittorrent to Sonarr:

1. Open Sonarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `qBittorrent` from the list
4. Configure the following:
   - **Name**: `qBittorrent` (or your preferred name)
   - **Enable**: `checked`
   - **Host**: the address where qBittorrent is reachable
     - If on the same Docker network: `qbittorrent:8080`
     - If using host IP: `<host-ip>:8080`
   - **Port**: `8080`
   - **Username**: `admin` (or your configured username)
   - **Password**: your configured qBittorrent password
   - **Category**: `sonarr` (recommended for organizing torrents)
   - **Priority**: `0` (default)
   - **Use SSL**: `unchecked` (unless you've configured SSL)
   - **Recent Priority**: `Normal` (or `High` for faster processing of recent episodes)
   - **Post-Import Category**: Leave empty (or set if you have post-import automation)
5. Click `Test` to verify Sonarr can contact qBittorrent
6. Click `Save` to add the download client

Once configured, Sonarr will send torrent files directly to qBittorrent for downloading. Completed downloads will appear in `/data/downloads/qbittorrent/completed` for post-processing.

### Add qBittorrent to Radarr:

1. Open Radarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `qBittorrent` from the list
4. Configure the following:
   - **Name**: `qBittorrent` (or your preferred name)
   - **Enable**: `checked`
   - **Host**: the address where qBittorrent is reachable
     - If on the same Docker network: `qbittorrent:8080`
     - If using host IP: `<host-ip>:8080`
   - **Port**: `8080`
   - **Username**: `admin` (or your configured username)
   - **Password**: your configured qBittorrent password
   - **Category**: `radarr` (recommended for organizing torrents)
   - **Priority**: `0` (default)
   - **Use SSL**: `unchecked` (unless you've configured SSL)
   - **Recent Priority**: `Normal` (or `High` for faster processing of recent releases)
   - **Post-Import Category**: Leave empty (or set if you have post-import automation)
5. Click `Test` to verify Radarr can contact qBittorrent
6. Click `Save` to add the download client

Once configured, Radarr will send torrent files directly to qBittorrent for downloading. Completed downloads will appear in `/data/downloads/qbittorrent/completed` for post-processing.

## 7. Manage Categories for Organization (Recommended)

Create categories in qBittorrent to organize torrents by source application and improve post-processing:

1. Open qBittorrent web UI -> `Edit` -> `Preferences` or right-click on torrent list
2. Alternatively, use `Tools` -> `Options` -> `Downloads` -> set your save paths
3. Add or configure categories:
   - In the left sidebar of the main qBittorrent UI, right-click `Categories` or `All`
   - Create new categories: `sonarr`, `radarr`, etc.
   - Assign a save path to each category (e.g., `/data/torrents/completed` for both)

When adding qBittorrent to Sonarr/Radarr, specify the category so downloads are organized:
- Sonarr torrents go to category `sonarr`
- Radarr torrents go to category `radarr`

This helps with monitoring, organization, and potential per-application seeding limits.

## 8. Monitor and Maintain

### Regular Monitoring:
- Check the qBittorrent web UI regularly to monitor download progress
- Watch for stalled or failed torrents and remove them
- Monitor seeding status to ensure torrents honor your seeding limits

### Performance Tuning:
- Adjust `Max Active Downloads` based on your bandwidth
- If downloads are slow, increase connection limits in `Tools` -> `Options` -> `Bittorrent`
- Monitor memory usage; restart if needed with `docker-compose restart qbittorrent`

### Maintenance:
- Periodically clear old torrent history: `Tools` -> `Options` -> `BitTorrent` -> increase Inactive Seeding Limit
- Monitor disk space in `/data/downloads/qbittorrent/completed`
- Update qBittorrent container regularly: `docker-compose pull && docker-compose up -d qbittorrent`

## 9. Download Flow

```
1. Sonarr/Radarr finds a match
   ↓
2. Sends torrent file to qBittorrent (category: sonarr/radarr)
   ↓
3. qBittorrent queues the torrent
   ↓
4. Download starts; active download stored in /data/downloads/qbittorrent/incomplete
   ↓
5. Download completes
   ↓
6. Move to /data/downloads/qbittorrent/completed
   ↓
7. Sonarr/Radarr detects and post-processes
   ↓
8. Files organized to /data/shows or /data/movies
   ↓
9. Seeding continues per configured limits (ratio/time)
   ↓
10. Jellyfin scans and adds to library
```

## Troubleshooting

### qBittorrent cannot connect to Sonarr/Radarr:
- Verify network connectivity: Both services must be on the same Docker network or reachable via host IP
- Check credentials: Verify username and password in Sonarr/Radarr settings
- Test connection: Click the `Test` button in Sonarr/Radarr download client settings
- Check firewall: Port 8080 must be accessible from Sonarr/Radarr containers

### No peer connections (slow or stalled downloads):
- Verify DHT, PEX, and LSD are enabled in `Tools` -> `Options` -> `Bittorrent`
- Check listening port: Ensure port 6881 (or configured port) is accessible
- Verify no double NAT or restrictive firewall is blocking peer connections
- Try a different torrent to rule out tracker issues

### Downloads not appearing in completed folder:
- Verify save paths in `Tools` -> `Options` -> `Downloading`
- Check `/data/downloads/qbittorrent/completed` folder exists and has proper permissions
- Verify category-specific save paths are configured
- Check qBittorrent logs for permission or path errors: `docker-compose logs -f qbittorrent`

### High memory usage:
- Reduce `Max Active Torrents` in `Tools` -> `Options` -> `Bittorrent`
- Lower `Max Active Downloads` in qBittorrent settings
- Consider increasing memory limit in compose.yaml if needed

### Torrents stuck in seeding:
- Check seeding limits in `Tools` -> `Options` -> `Bittorrent`
- Verify ratio and time limits are configured correctly
- Manually remove stalled torrents from the UI
- Check that `Complete` folder is correct and downloads are actually there

### Connection issues after Docker restart:
- qBittorrent may take 10-30 seconds to initialize
- Wait before attempting connections from Sonarr/Radarr
- Restart Sonarr/Radarr if they show connection failures
- Check logs: `docker-compose logs qbittorrent`
