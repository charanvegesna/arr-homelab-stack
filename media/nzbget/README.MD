# NZBGet Install and Docker-Compose Guide

NZBGet is a usenet downloader for Linux/Unix systems. It's an efficient alternative to SABnzbd+ or other usenet clients and works as a backend service for Sonarr and Radarr to automatically download completed requests.

## Directory Structure

NZBGet uses the following directory structure (defined in the main media README):

```
/data/downloads/nzbget
├── /completed               # Completed usenet downloads (ready for post-processing)
└── /intermediate            # In-progress usenet downloads (being downloaded)
```

### Create NZBGet Directories

Before starting NZBGet, create the required directories:

```bash
# Create NZBGet download directories
mkdir -p /data/downloads/nzbget/completed
mkdir -p /data/downloads/nzbget/intermediate

# Set proper permissions (adjust 1000:1000 to your PGID:PUID)
chown -R 1000:1000 /data/downloads/nzbget
find /data/downloads/nzbget -type d -exec chmod 755 {} \;
```

Or use this one-liner:
```bash
mkdir -p /data/downloads/nzbget/{completed,intermediate} && chown -R 1000:1000 /data/downloads/nzbget
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add NZBGet service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host. This snippet intentionally removes any personal or machine-specific absolute paths.

```yaml
nzbget:
  cpu_shares: 90
  command: []
  container_name: nzbget
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"  # replace with your timezone
  hostname: nzbget
  image: lscr.io/linuxserver/nzbget:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 6789
      published: "6789"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/nzbget  # replace with your config path
      target: /config
      bind:
        create_host_path: true
    - type: bind
      source: /data/downloads/nzbget/completed  # Completed downloads directory
      target: /completed
      bind:
        create_host_path: true
    - type: bind
      source: /data/downloads/nzbget/intermediate  # In-progress downloads directory
      target: /intermediate
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/nzbget` with your config storage path (typically `/docker/servarr/nzbget`).
- `/data/downloads/nzbget/completed` - where finished NZB downloads are stored (Sonarr/Radarr collect from here)
- `/data/downloads/nzbget/intermediate` - where active downloads are stored temporarily
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- The config volume stores NZBGet's configuration, database, and download history.
- **Important:** Both download directories must exist before NZBGet starts to avoid permission issues.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d nzbget
```

## 4. Verify

- Check logs: `docker compose logs -f nzbget`
- Visit `http://<host-ip>:6789` to access the NZBGet web UI (replace `<host-ip>` with the server IP).
- You should see the NZBGet control interface on first access.

## 5. Connect NZBGet to Sonarr and Radarr

NZBGet acts as a download client for Sonarr and Radarr. Both applications will send NZB files to NZBGet for downloading.

### Prerequisites:
- Sonarr and Radarr must be running and reachable from the NZBGet host (same Docker network or host IP and port).
- You will need the NZBGet API key and the host:port where it is reachable (NZBGet default port: `6789`).
- A valid usenet provider account is required for downloading (NZBGet connects to usenet servers).

### Get the NZBGet credentials:
- Open NZBGet web UI at `http://<host-ip>:6789`
- Navigate to `Settings` -> `Security`
- Copy the `API Key` (or use default username/password if not changed)

### Add NZBGet to Sonarr:

1. Open Sonarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `NZBGet` from the list
4. Configure the following:
   - Name: `NZBGet` (or your preferred name)
   - Enable: `checked`
   - Host: use `nzbget:6789` if on the same Docker network, or `<host-ip>:6789`
   - Port: `6789`
   - API Key: paste the API key from NZBGet
   - Username: (leave blank if using API key)
   - Password: (leave blank if using API key)
   - Category: `sonarr` (optional but recommended for organization)
   - Priority: `0` (default)
   - Use SSL: `unchecked` (unless you've configured SSL)
5. Click `Test` to verify Sonarr can contact NZBGet
6. Click `Save` to add the download client

### Add NZBGet to Radarr:

1. Open Radarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `NZBGet` from the list
4. Configure the following:
   - Name: `NZBGet` (or your preferred name)
   - Enable: `checked`
   - Host: use `nzbget:6789` if on the same Docker network, or `<host-ip>:6789`
   - Port: `6789`
   - API Key: paste the API key from NZBGet
   - Username: (leave blank if using API key)
   - Password: (leave blank if using API key)
   - Category: `radarr` (optional but recommended for organization)
   - Priority: `0` (default)
   - Use SSL: `unchecked` (unless you've configured SSL)
5. Click `Test` to verify Radarr can contact NZBGet
6. Click `Save` to add the download client

### Configure NZBGet for usenet:

Before downloads will work, you need to configure NZBGet with your usenet provider:

1. Open NZBGet web UI -> `Settings` -> `Server`
2. Add your usenet server details:
   - Host: your usenet provider's server address
   - Port: typically `119` (unencrypted) or `563` (SSL)
   - Username: your usenet account username
   - Password: your usenet account password
   - Use SSL: check if required by your provider
   - Connections: set to your provider's allowed connections (typical range: 4-8)
3. Click `Test` to verify connection to your usenet server
4. Save the settings

### Configure NZBGet Download Folders:

Configure where NZBGet stores downloads (must match your volume mounts):

1. Open NZBGet web UI -> `Settings` -> `Paths`
2. Set the following paths:
   - **MainDir** (working directory): `/intermediate` (matches the container volume mount)
   - **InterDir** (intermediate directory): `/intermediate/temp` or leave default
   - **CompleteDir** (complete directory): `/completed` (matches the container volume mount)
3. Click `Save` to apply settings

This ensures downloaded files are placed in the correct directories that Sonarr/Radarr can access.

Once configured, Sonarr and Radarr will automatically send NZB files to NZBGet for downloading from usenet. Completed downloads will appear in `/data/downloads/nzbget/completed` for post-processing.

## Download Flow

```
1. Sonarr/Radarr finds a match
   ↓
2. Sends NZB file to NZBGet
   ↓
3. NZBGet downloads from usenet server
   ↓
4. Active download stored in /data/downloads/nzbget/intermediate
   ↓
5. Download completes
   ↓
6. Move to /data/downloads/nzbget/completed
   ↓
7. Sonarr/Radarr detects and post-processes
   ↓
8. Files organized to /data/shows or /data/movies
   ↓
9. Jellyfin scans and adds to library
```

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d nzbget
```

## 4. Verify

- Check logs: `docker compose logs -f nzbget`
- Visit `http://<host-ip>:6789` to access the NZBGet web UI (replace `<host-ip>` with the server IP).
- You should see the NZBGet control interface on first access.

## 5. Connect NZBGet to Sonarr and Radarr

NZBGet acts as a download client for Sonarr and Radarr. Both applications will send NZB files to NZBGet for downloading.

### Prerequisites:
- Sonarr and Radarr must be running and reachable from the NZBGet host (same Docker network or host IP and port).
- You will need the NZBGet API key and the host:port where it is reachable (NZBGet default port: `6789`).
- A valid usenet provider account is required for downloading (NZBGet connects to usenet servers).

### Get the NZBGet credentials:
- Open NZBGet web UI at `http://<host-ip>:6789`
- Navigate to `Settings` -> `Security`
- Copy the `API Key` (or use default username/password if not changed)

### Add NZBGet to Sonarr:

1. Open Sonarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `NZBGet` from the list
4. Configure the following:
   - Name: `NZBGet` (or your preferred name)
   - Enable: `checked`
   - Host: the hostname or IP where NZBGet is reachable (use `nzbget:6789` if on the same Docker network, or `<host-ip>:6789`)
   - Port: `6789`
   - API Key: paste the API key from NZBGet
   - Username: (leave blank if using API key)
   - Password: (leave blank if using API key)
   - Category: `sonarr` (optional but recommended for organization)
   - Priority: `0` (default)
   - Use SSL: `unchecked` (unless you've configured SSL)
5. Click `Test` to verify Sonarr can contact NZBGet
6. Click `Save` to add the download client

### Add NZBGet to Radarr:

1. Open Radarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `NZBGet` from the list
4. Configure the following:
   - Name: `NZBGet` (or your preferred name)
   - Enable: `checked`
   - Host: the hostname or IP where NZBGet is reachable (use `nzbget:6789` if on the same Docker network, or `<host-ip>:6789`)
   - Port: `6789`
   - API Key: paste the API key from NZBGet
   - Username: (leave blank if using API key)
   - Password: (leave blank if using API key)
   - Category: `radarr` (optional but recommended for organization)
   - Priority: `0` (default)
   - Use SSL: `unchecked` (unless you've configured SSL)
5. Click `Test` to verify Radarr can contact NZBGet
6. Click `Save` to add the download client

### Configure NZBGet for usenet:

Before downloads will work, you need to configure NZBGet with your usenet provider:

1. Open NZBGet web UI -> `Settings` -> `Server`
2. Add your usenet server details:
   - Host: your usenet provider's server address
   - Port: typically `119` (unencrypted) or `563` (SSL)
   - Username: your usenet account username
   - Password: your usenet account password
   - Use SSL: check if required by your provider
   - Connections: set to your provider's allowed connections (typical range: 4-8)
3. Click `Test` to verify connection to your usenet server
4. Save the settings

Once configured, Sonarr and Radarr will automatically send NZB files to NZBGet for downloading from usenet.
