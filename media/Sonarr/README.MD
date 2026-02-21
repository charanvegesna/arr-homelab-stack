# Sonarr Install and Docker-Compose Guide

Sonarr is a PVR (Personal Video Recorder) for Usenet and BitTorrent users. It monitors RSS feeds and indexers for new episodes of your favorite TV shows, automatically downloads them using your configured download clients, and organizes files with proper naming conventions. It supports quality upgrades, episode monitoring, and integration with Prowlarr for centralized indexer management.

## Directory Structure

Sonarr uses the following directory structure (defined in the main media README):

```
/docker/servarr/sonarr
└── /config                 # Sonarr configuration, database, and settings

/data/media
└── /shows                  # TV show library storage (organized by Sonarr)

/data/downloads/combined
├── /completed              # Completed downloads (from all download clients)
└── /incomplete             # In-progress downloads
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Sonarr service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host. This snippet intentionally removes any personal or machine-specific absolute paths.

```yaml
sonarr:
  cpu_shares: 90
  command: []
  container_name: sonarr
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"  # replace with your timezone
  hostname: sonarr
  image: lscr.io/linuxserver/sonarr:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 8989
      published: "8989"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/sonarr  # replace with your config path
      target: /config
      bind:
        create_host_path: true
    - type: bind
      source: /data/media/shows  # TV show library directory
      target: /shows
      bind:
        create_host_path: true
    - type: bind
      source: /data/downloads/combined  # Shared downloads directory
      target: /downloads
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/sonarr` with your config storage path (typically `/docker/servarr/sonarr`).
- `/data/media/shows` - where Sonarr will organize and store your TV show library
- `/data/downloads/combined` - shared downloads directory (used by both download clients)
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- The config volume stores Sonarr's database, settings, indexer configurations, and download history.
- The shows volume should point to where your TV show library will be organized.
- The downloads volume should point to where your download clients save completed files.
- Docker Compose will automatically create all required directories with the `create_host_path: true` setting.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d sonarr
```

## 4. Verify

- Check logs: `docker compose logs -f sonarr`
- Visit `http://<host-ip>:8989` to access the Sonarr web UI (replace `<host-ip>` with the server IP).
- You will be guided through the initial setup wizard on first access.

## 5. Initial Setup

When you first access the Sonarr web UI, perform these essential configuration steps:

1. Open `http://<host-ip>:8989` in your browser
2. You will be guided through the initial setup wizard on first access
3. After setup, configure the following critical settings:

### Configure Media Management:

1. Open Sonarr -> `Settings` -> `Media Management`
   - **Rename Episodes**: Enable `checked` (Sonarr will organize files with standardized naming)
   - **Series Folder Format**: Set format for series directories (default: `{Series Title}`)
   - **Season Folder Format**: Set format for season directories (default: `Season {season:00}`)
   - **Episode Folder Format**: Set format for episode naming
   - **Root Folders**: Configure or add your TV shows directory

### Add Root Folders:

1. Open Sonarr -> `Settings` -> `Media Management` -> `Root Folders`
2. Click `Add Root Folder`
3. Set the path to your TV shows directory:
   - Path: `/shows` (matches your container volume mount)
   - Monitor: `All Folders` (default)
4. Click the checkmark to add
5. This folder will be the default location for organizing downloaded episodes

### Configure Download Clients Handling:

1. Open Sonarr -> `Settings` -> `Download Clients`
   - Verify your download clients are added (see section 6 below)
   - **Completed Download Handling**: Enable `checked`
   - **Remove**: `checked` (remove torrents/NZBs after import)
   - **Redownload**: Enable for automatic quality upgrades

### Configure Import Settings:

1. Open Sonarr -> `Settings` -> `Importers`
   - Configure any custom lists for automatic series imports
   - Set up quality profiles for desired episode formats

## 6. Connect Prowlarr (Indexer Management)

Prowlarr provides centralized indexer management for Sonarr.

### Prerequisites:
- Prowlarr must be running and reachable from the Sonarr host (same Docker network or host IP and port).
- You will need the Prowlarr API key.

### Add Prowlarr to Sonarr:

1. Open Sonarr web UI -> `Settings` -> `Indexers`
2. Click the settings icon (gear icon) in the top right
3. Navigate to the `Prowlarr` tab
4. Configure the following:
   - **Enable**: `checked`
   - **Prowlarr Server**: the address where Prowlarr is reachable
     - If on the same Docker network: `http://prowlarr:9696`
     - If using host IP: `http://<host-ip>:9696`
   - **API Key**: paste your Prowlarr API key (found in Prowlarr -> `Settings` -> `General` -> Security section)
   - **Sync Interval**: `12` hours (default; adjust for more frequent syncs)
   - **Categories**: Select TV-related categories if available
5. Click `Test` to verify Sonarr can contact Prowlarr
6. Click `Save`

Once configured, Sonarr will automatically fetch available indexers from Prowlarr for searching and downloading TV episodes.

## 7. Connect Download Clients

Sonarr needs at least one download client to automatically download TV shows. You can use qBittorrent for torrents, NZBGet for usenet, or both.

### Add qBittorrent (Torrent Downloads):

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
   - **Password**: your qBittorrent password
   - **Category**: `sonarr` (recommended for organizing torrents)
   - **Priority**: `0` (default)
   - **Use SSL**: `unchecked` (unless configured)
   - **Post-Import Category**: Leave empty
5. Click `Test` to verify connection
6. Click `Save`

Once added, Sonarr will send torrent files to qBittorrent for downloading.

### Add NZBGet (Usenet Downloads):

1. Open Sonarr -> `Settings` -> `Download Clients`
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
   - **Category**: `sonarr` (recommended for organizing downloads)
   - **Priority**: `0` (default)
   - **Use SSL**: `unchecked` (unless configured)
5. Click `Test` to verify connection
6. Click `Save`

Once added, Sonarr will send NZB files to NZBGet for downloading from usenet.

## 8. Configure Quality Profiles

Quality profiles define the quality and format of TV episodes Sonarr should download.

1. Open Sonarr -> `Settings` -> `Profiles` -> `Quality`
2. Review or create quality profiles:
   - **Default Profile**: Uses sensible defaults (Bluray, HDTV, etc.)
   - **Custom Profiles**: Create profiles for specific quality needs
3. Each profile defines:
   - Minimum quality to grab
   - Preferred quality
   - Upgrade until quality is reached
4. Assign profiles when adding shows to control their download quality

## 9. Add and Manage TV Shows

Once setup is complete, you can start adding TV shows:

### Add TV Shows Manually:

1. Open Sonarr -> `Add Series` or `Series` menu
2. Search for a TV show title
3. Select the desired show from results
4. Choose your preferred language profile and quality profile
5. Set the root folder where the show will be stored
6. Click `Add Series`
7. Sonarr will search Prowlarr indexers for available episodes
8. When matches are found, they automatically send to your download client
9. Once downloaded and processed, episodes are moved to `/data/media/shows` with proper naming

### Series States:
- **Monitored**: Actively searching for new episodes
- **Unmonitored**: In library but not searching
- **Continuing**: Show is still airing
- **Ended**: Show has completed
- **Missing**: Episode available to download but not yet downloaded

### Configure Series Monitoring:

Control which episodes Sonarr monitors and searches for:

1. Open Sonarr -> `Series`
2. Click on a series to edit it
3. Set monitoring options:
   - **Monitor**: Select which seasons/episodes to monitor (All, Future, Only Latest, etc.)
   - **Cutoff Unmet**: Automatically search for upgrades to better quality
4. Click `Save Changes`

Different monitoring profiles allow you to:
- Monitor all existing episodes
- Monitor only new episodes
- Monitor upcoming releases only
- Custom selection of seasons

### Automatic Quality Upgrades:
- Sonarr automatically searches for higher quality versions of existing episodes
- Configure upgrade limits in quality profiles to control behavior

## 10. Connect to Jellyseerr (Optional)

Jellyseerr allows users to request TV shows that will be automatically added to Sonarr for download.

### Prerequisites:
- Jellyseerr must be running and reachable from the Sonarr host.

### Configure Jellyseerr:

1. Open Jellyseerr web UI at `http://<host-ip>:5055`
2. During setup wizard, configure Sonarr connection:
   - **Host**: the address where Sonarr is reachable
     - If on the same Docker network: `sonarr:8989`
     - If using host IP: `<host-ip>:8989`
   - **API Key**: found in Sonarr -> `Settings` -> `General` -> `Security` -> `API Key`
   - **Root Folder**: select the root folder you configured for TV shows (`/shows`)
   - **Quality Profile**: select your preferred quality profile
   - **Language Profile**: select your preferred language settings
3. Click `Test` to verify connection
4. Save settings

Once configured, users can request TV shows through Jellyseerr and they will be automatically added to Sonarr for download.

## 11. Connect to Bazarr (Optional)

Bazarr automatically downloads subtitles for your TV shows:

### Prerequisites:
- Bazarr must be running and reachable from the Sonarr host.

### Configure Bazarr:

1. Open Bazarr web UI at `http://<host-ip>:6767`
2. Navigate to `Settings` -> `Services` -> `Sonarr`
3. Configure the following:
   - **Host**: the address where Sonarr is reachable
     - If on the same Docker network: `sonarr:8989`
     - If using host IP: `<host-ip>:8989`
   - **API Key**: found in Sonarr -> `Settings` -> `General` -> `Security` -> `API Key`
   - **Base URL**: leave blank unless Sonarr is behind a reverse proxy
4. Click `Test` to verify connection
5. Save settings

Once configured, Bazarr will automatically download subtitles for new episodes as they are added to Sonarr.

## 12. Monitor and Manage

### Dashboard:
- Check the main dashboard for upcoming episodes and download progress
- View queued downloads and import status
- Monitor indexer health and availability

### Series Library:
- **Series Menu**: View full TV show library with quality and monitored status indicators
- **Wanted**: Episodes actively being searched for (will download when available)
- **Missing**: Episodes available to watch but not yet downloaded
- **Future**: Upcoming episode releases
- **Cutoff**: Episodes available but at lower quality than profile allows
- **Calendar**: Calendar view of upcoming releases

### History and Logs:
- **History**: Review past downloads and automatic upgrades
- **Logs**: View application logs for troubleshooting
- **Settings**: Adjust profiles, delays, and other preferences as needed

### Regular Maintenance:
- Monitor download client status (qBittorrent/NZBGet)
- Check indexer connectivity through Prowlarr
- Verify disk space in `/data/media/shows`
- Update Sonarr container: `docker-compose pull && docker-compose up -d sonarr`

Sonarr will continuously monitor your configured indexers through Prowlarr and automatically download episodes based on your quality profiles, monitored status, and availability.

## 13. Episode Download Workflow

```
1. TV show added to Sonarr (manually or via Jellyseerr)
   ↓
2. New episodes detected (via RSS feed or manual search)
   ↓
3. Sonarr searches Prowlarr indexers
   ↓
4. Match found; NZB or torrent sent to download client
   ↓
5. qBittorrent/NZBGet downloads episode
   ↓
6. Download stored in /data/downloads/combined/completed
   ↓
7. Sonarr detects completed download
   ↓
8. Post-processing: Extract, verify, quality check
   ↓
9. Rename/organize to /data/media/shows/{Series Title}/Season {season}/
   ↓
10. Bazarr downloads subtitles (if configured)
   ↓
11. Jellyfin scans and adds to library
   ↓
12. Episode available in Jellyfin for streaming
```

## 14. Optimize Performance

For better performance with many shows:

1. Open Sonarr -> `Settings` -> `General` -> `Updates`
2. Set `Automatic` to check for new episodes at reasonable intervals (e.g., every 6-12 hours)
3. Adjust RSS Sync Interval in `Settings` -> `Indexers` (default: every 15 minutes)
4. Configure delays in `Settings` -> `Download Clients` to avoid duplicate downloads during early release windows
5. Set proper file permissions to ensure post-processing works correctly

This will help Sonarr efficiently manage your TV show library without overwhelming your system resources.

## 15. Troubleshooting

### Episodes not downloading:
- Verify Prowlarr is connected and indexers are enabled
- Check download clients are added and connected
- Verify TV shows are marked as Monitored in Sonarr
- Check Sonarr logs for search/download errors: `docker-compose logs -f sonarr`
- Ensure quality profiles allow available releases
- Verify episode cutoff isn't set too high

### Episodes not importing after download:
- Verify download paths match between download client and Sonarr
- Check permissions on `/data/downloads/combined` and `/data/media/shows`
- Ensure Completed Download Handling is enabled in settings
- Monitor Sonarr logs for import errors
- Verify file naming is recognized (check episode naming format)

### Failed to connect to Prowlarr:
- Verify Prowlarr URL is correct (use `http://prowlarr:9696` if on same network)
- Check API key in Sonarr settings matches Prowlarr
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
- Monitor `/data/media/shows` for proper organization
- Clean up old/failed downloads manually
- Consider increasing disk size or archiving old series

### Episodes stuck as Missing:
- Manually trigger search: Click series -> `Search All` or search specific episode
- Verify quality profile allows available releases
- Check indexer availability through Prowlarr
- Increase search frequency if needed
- Check if releases exist for that show/quality/language combination

### Subtitle download issues (Bazarr):
- Verify Bazarr is configured and connected
- Check Bazarr logs for subtitle provider issues
- Ensure subtitle providers are enabled in Bazarr
- Verify show/episode is properly added to the library

### Performance issues with large library:
- Consider disabling automatic searches and use RSS sync instead
- Reduce `Max Active Downloads` in download clients
- Monitor Sonarr memory usage: increase in compose.yaml if needed
- Restart Sonarr periodically: `docker-compose restart sonarr`
