# Radarr Install and Docker-Compose Guide

Radarr is a movie collection manager and organizer for Usenet and BitTorrent users. It monitors RSS feeds and indexers for new movies, automatically downloads them using your configured download clients, and organizes files with proper naming conventions. It supports quality upgrades, indexer and download client management through integration with Prowlarr and other services.

## Directory Structure

Radarr uses the following directory structure (defined in the main media README):

```
/docker/servarr/radarr
└── /config                 # Radarr configuration, database, and settings

/data/media
└── /movies                 # Movie library storage (organized by Radarr)

/data/downloads/combined
├── /completed              # Completed downloads (from all download clients)
└── /incomplete             # In-progress downloads
```

### Create Radarr Directories

Before starting Radarr, create the required directories:

```bash
# Create Radarr config directory
mkdir -p /docker/servarr/radarr

# Create movie library directory
mkdir -p /data/media/movies

# Create shared download directories
mkdir -p /data/downloads/combined/completed
mkdir -p /data/downloads/combined/incomplete

# Set proper permissions (adjust 1000:1000 to your PGID:PUID)
chown -R 1000:1000 /docker/servarr/radarr /data/media/movies /data/downloads/combined
find /docker/servarr/radarr -type d -exec chmod 755 {} \;
find /data/media/movies -type d -exec chmod 755 {} \;
find /data/downloads/combined -type d -exec chmod 755 {} \;
```

Or use this one-liner:
```bash
mkdir -p /docker/servarr/radarr /data/media/movies /data/downloads/combined/{completed,incomplete} && chown -R 1000:1000 /docker/servarr/radarr /data/media/movies /data/downloads/combined
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Radarr service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host. This snippet intentionally removes any personal or machine-specific absolute paths.

```yaml
radarr:
  cpu_shares: 90
  command: []
  container_name: radarr
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"  # replace with your timezone
  hostname: radarr
  image: lscr.io/linuxserver/radarr:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 7878
      published: "7878"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/radarr  # replace with your config path
      target: /config
      bind:
        create_host_path: true
    - type: bind
      source: /data/media/movies  # Movie library directory
      target: /movies
      bind:
        create_host_path: true
    - type: bind
      source: /data/downloads/combined  # Shared downloads directory
      target: /downloads
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/radarr` with your config storage path (typically `/docker/servarr/radarr`).
- `/data/media/movies` - where Radarr will organize and store your movie library
- `/data/downloads/combined` - shared downloads directory (used by both download clients)
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- The config volume stores Radarr's database, settings, indexer configurations, and download history.
- The movies volume should point to where your movie library will be organized.
- The downloads volume should point to where your download clients save completed files.
- **Important:** All directories must exist before Radarr starts to avoid permission issues.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d radarr
```

## 4. Verify

- Check logs: `docker compose logs -f radarr`
- Visit `http://<host-ip>:7878` to access the Radarr web UI (replace `<host-ip>` with the server IP).
- You will be guided through the initial setup wizard on first access.

## 5. Initial Setup

When you first access the Radarr web UI, perform these essential configuration steps:

1. Open `http://<host-ip>:7878` in your browser
2. You will be guided through the initial setup wizard on first access
3. After setup, configure the following critical settings:

### Configure Media Management:

1. Open Radarr -> `Settings` -> `Media Management`
   - **Rename Movies**: Enable `checked` (Radarr will organize files with standardized naming)
   - **Movie Folder Format**: Set format for movie directories (default: `{Movie CleanTitle} ({Release Year})`)
   - **Movie File Format**: Set format for movie files
   - **Root Folders**: Configure or add your movie directory

### Add Root Folders:

1. Open Radarr -> `Settings` -> `Media Management` -> `Root Folders`
2. Click `Add Root Folder`
3. Set the path to your movies directory:
   - Path: `/movies` (matches your container volume mount)
   - Monitor: `All Folders` (default)
4. Click the checkmark to add
5. This folder will be the default location for organizing downloaded movies

### Configure Download Clients Handling:

1. Open Radarr -> `Settings` -> `Download Clients`
   - Verify your download clients are added (see section 6 below)
   - **Completed Download Handling**: Enable `checked`
   - **Remove**: `checked` (remove torrents/NZBs after import)
   - **Redownload**: Enable for automatic quality upgrades

### Configure Import Settings:

1. Open Radarr -> `Settings` -> `Importers`
   - Configure any custom lists for automatic movie imports
   - Set up quality profiles for desired movie formats

## 6. Connect Prowlarr (Indexer Management)

Prowlarr provides centralized indexer management for Radarr.

### Prerequisites:
- Prowlarr must be running and reachable from the Radarr host (same Docker network or host IP and port).
- You will need the Prowlarr API key.

### Add Prowlarr to Radarr:

1. Open Radarr web UI -> `Settings` -> `Indexers`
2. Click the settings icon (gear icon) in the top right
3. Navigate to the `Prowlarr` tab
4. Configure the following:
   - **Enable**: `checked`
   - **Prowlarr Server**: the address where Prowlarr is reachable
     - If on the same Docker network: `http://prowlarr:9696`
     - If using host IP: `http://<host-ip>:9696`
   - **API Key**: paste your Prowlarr API key (found in Prowlarr -> `Settings` -> `General` -> Security section)
   - **Sync Interval**: `12` hours (default; adjust for more frequent syncs)
   - **Categories**: Select movie-related categories if available
5. Click `Test` to verify Radarr can contact Prowlarr
6. Click `Save`

Once configured, Radarr will automatically fetch available indexers from Prowlarr for searching and downloading movies.

## 7. Connect Download Clients

Radarr needs at least one download client to automatically download movies. You can use qBittorrent for torrents, NZBGet for usenet, or both.

### Add qBittorrent (Torrent Downloads):

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
   - **Password**: your qBittorrent password
   - **Category**: `radarr` (recommended for organizing torrents)
   - **Priority**: `0` (default)
   - **Use SSL**: `unchecked` (unless configured)
   - **Post-Import Category**: Leave empty
5. Click `Test` to verify connection
6. Click `Save`

Once added, Radarr will send torrent files to qBittorrent for downloading.

### Add NZBGet (Usenet Downloads):

1. Open Radarr -> `Settings` -> `Download Clients`
2. Click the `+` button to add a new download client
3. Select `NZBGet` from the list
4. Configure the following:
   - **Name**: `NZBGet` (or your preferred name)
   - **Enable**: `checked`
   - **Host**: the address where NZBGet is reachable
     - If on the same Docker network: `nzbget:6789`
     - If using host IP: `<host-ip>:6789`
   - **Port**: `6789`
   - **API Key**: your NZBGet API key (found in NZBGet -> `Settings` -> `Security`)
   - **Username**: Leave blank if using API key
   - **Password**: Leave blank if using API key
   - **Category**: `radarr` (recommended for organizing downloads)
   - **Priority**: `0` (default)
   - **Use SSL**: `unchecked` (unless configured)
5. Click `Test` to verify connection
6. Click `Save`

Once added, Radarr will send NZB files to NZBGet for downloading from usenet.

## 8. Configure Quality Profiles

Quality profiles define the quality and format of movies Radarr should download.

1. Open Radarr -> `Settings` -> `Profiles` -> `Quality`
2. Review or create quality profiles:
   - **Default Profile**: Uses sensible defaults (Bluray, HDTV, etc.)
   - **Custom Profiles**: Create profiles for specific quality needs
3. Each profile defines:
   - Minimum quality to grab
   - Preferred quality
   - Upgrade until quality is reached
4. Assign profiles when adding movies to control their download quality

## 9. Connect to Jellyseerr (Optional)

Jellyseerr allows users to request movies that will be automatically added to Radarr for download.

### Prerequisites:
- Jellyseerr must be running and reachable from the Radarr host.

### Configure Jellyseerr:

1. Open Jellyseerr web UI at `http://<host-ip>:5055`
2. During setup wizard, configure Radarr connection:
   - **Host**: the address where Radarr is reachable
     - If on the same Docker network: `radarr:7878`
     - If using host IP: `<host-ip>:7878`
   - **API Key**: found in Radarr -> `Settings` -> `General` -> `Security` -> `API Key`
   - **Root Folder**: select the root folder you configured for movies (`/movies`)
   - **Quality Profile**: select your preferred quality profile
3. Click `Test` to verify connection
4. Save settings

Once configured, users can request movies through Jellyseerr and they will be automatically added to Radarr for download.

## 10. Add and Manage Movies

Once setup is complete, you can start adding movies:

### Add Movies Manually:

1. Open Radarr web UI -> `Add Movies`
2. Search for a movie title
3. Select the desired movie from results
4. Choose quality profile and root folder
5. Click `Add Movie`
6. Radarr will search Prowlarr indexers for available downloads
7. When a match is found, it automatically sends the NZB or torrent to your download client
8. Once downloaded and processed, the movie is moved to `/data/media/movies` with proper naming

### Movie States:
- **Missing**: Available to watch but not yet downloaded
- **Monitored**: Actively searching for releases
- **Unmonitored**: In library but not searching
- **Downloaded**: Successfully imported into library

### Automatic Quality Upgrades:
- Radarr automatically searches for higher quality versions of existing movies
- Configure upgrade limits in quality profiles to control behavior

## 11. Monitor and Manage

### Dashboard:
- Check the main dashboard for upcoming releases and download progress
- View queued downloads and import status
- Monitor indexer health and availability

### Movie Library:
- **Movies Menu**: View full movie library with quality and selection indicators
- **Missing**: See movies you're monitoring that haven't been downloaded yet
- **Wanted**: Movies actively being searched for (will download when available)
- **Future**: Upcoming movie releases
- **Cutoff**: Movies available but at lower quality than profile allows

### History and Logs:
- **History**: Review past downloads and automatic upgrades
- **Logs**: View application logs for troubleshooting
- **Settings**: Adjust profiles, delays, and other preferences as needed

### Regular Maintenance:
- Monitor download client status (qBittorrent/NZBGet)
- Check indexer connectivity through Prowlarr
- Verify disk space in `/data/media/movies`
- Update Radarr container: `docker-compose pull && docker-compose up -d radarr`

Radarr will continuously monitor your configured indexers through Prowlarr and automatically download movies based on your quality profiles, monitored status, and availability.

## 12. Movie Organization Workflow

```
1. Movies added to Radarr (manually or via Jellyseerr)
   ↓
2. Radarr searches Prowlarr indexers
   ↓
3. Match found; NZB or torrent sent to download client
   ↓
4. qBittorrent/NZBGet downloads file
   ↓
5. Download stored in /data/downloads/combined/completed
   ↓
6. Radarr detects completed download
   ↓
7. Post-processing: Extract, verify, quality check
   ↓
8. Rename/organize to /data/media/movies/{Movie Name (Year)}/
   ↓
9. Jellyfin scans and adds to library
   ↓
10. Movie available in Jellyfin for streaming
```

## Troubleshooting

### Movies not downloading:
- Verify Prowlarr is connected and indexers are enabled
- Check download clients are added and connected
- Verify movies are marked as Monitored in Radarr
- Check Radarr logs for search/download errors: `docker-compose logs -f radarr`
- Ensure quality profiles allow available releases

### Movies not importing after download:
- Verify download paths match between download client and Radarr
- Check permissions on `/data/downloads/combined` and `/data/media/movies`
- Ensure Completed Download Handling is enabled in settings
- Monitor Radarr logs for import errors
- Verify file naming is recognized (check file format)

### Failed to connect to Prowlarr:
- Verify Prowlarr URL is correct (use `http://prowlarr:9696` if on same network)
- Check API key in Radarr settings matches Prowlarr
- Verify firewall allows port 9696
- Test connection button in settings

### Failed to connect to download clients:
- Verify download client host and port are correct
- Check credentials (username/password for qBittorrent, API key for NZBGet)
- Ensure download client is running: `docker-compose logs -f qbittorrent` or `nzbget`
- Verify networks are connected (same Docker network or host IP)
- Click Test button in download client settings

### High disk usage:
- Monitor `/data/downloads/combined` for stuck downloads
- Monitor `/data/media/movies` for proper organization
- Clean up old/failed downloads manually
- Consider increasing disk size or archiving old movies

### Movies stuck as Missing:
- Manually trigger search: Click movie -> `Search`
- Verify quality profile allows available releases
- Check indexer availability through Prowlarr
- Increase search frequency if needed
- Check if releases exist for that movie/quality combination
