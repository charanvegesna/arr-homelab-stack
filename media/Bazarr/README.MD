# Bazarr Install and Docker-Compose Guide

Bazarr is a subtitle management tool for Sonarr and Radarr. It automatically searches for, downloads, and manages subtitles for TV shows and movies in your library. Bazarr integrates seamlessly with both Sonarr and Radarr to provide a complete subtitle solution for your media collection.

## Directory Structure

Bazarr uses the following directory structure:

```
/docker/servarr/bazarr
└── /config                    # Bazarr configuration, database, and cache

/data/media
├── /shows                     # TV show library (mounted for subtitle scanning)
└── /movies                    # Movie library (mounted for subtitle scanning)
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Bazarr service to your compose file
- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host.

```yaml
bazarr:
  cpu_shares: 90
  command: []
  container_name: bazarr
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"           # replace with your timezone
  hostname: bazarr
  image: lscr.io/linuxserver/bazarr:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 6767
      published: "6767"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/bazarr   # Bazarr config directory
      target: /config
      bind:
        create_host_path: true
    - type: bind
      source: /data/media              # Media root for subtitle scanning
      target: /data
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/bazarr` with your config storage path (typically `/docker/servarr/bazarr`).
- Replace `/data/media` with your media root path (where your shows and movies directories are located).
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- The config volume stores Bazarr's configuration, database, and subtitle cache.
- The media volume allows Bazarr to scan and manage subtitles for your shows and movies.
- Docker Compose will automatically create all required directories with the `create_host_path: true` setting.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d bazarr
```

## 4. Verify

- Check logs: `docker compose logs -f bazarr`
- Check service is running: `docker compose ps bazarr`
- Visit `http://localhost:6767` to access the Bazarr web UI (replace `localhost` with your server IP if testing remotely)

## 5. Initial Setup and Configuration

### First Access and Language Configuration

When you first access Bazarr:

1. Open `http://<host-ip>:6767` in your browser
2. You'll be prompted to configure languages:
   - Click `Add Languages` to select subtitle languages you want to download
   - Select one or more languages (e.g., English, Spanish, French)
   - You can also select "Forced" subtitles (for foreign language scenes in primarily English content)
3. Configure these settings before connecting to Sonarr/Radarr

### Configure Subtitle Providers

Bazarr uses subtitle providers to search for subtitles. Configure providers in Settings:

1. Open Bazarr web UI -> `Settings` -> `Subtitles`
2. Enable at least one provider (popular providers):
   - **Opensubtitles.org**: Largest subtitle database
   - **TVsubtitles.net**: Good for TV shows
   - **Addic7ed**: High-quality TV show subtitles
   - **Shooter.cn**: Alternative provider
3. Configure API keys if required by the provider
4. Configure provider settings (language preferences, quality filtering, etc.)

## 6. Connect Bazarr to Sonarr and Radarr

Bazarr manages subtitles for content managed by Sonarr and Radarr.

### Prerequisites:
- Sonarr and Radarr must be running and reachable from the Bazarr host (same Docker network or host IP and port)
- You will need each service's API key (Settings -> General -> API Key)
- Sonarr default port: `8989`, Radarr default port: `7878`
- Media files must be in the directories where Bazarr can access them via `/data/media` volume

### Get the API keys:

**Sonarr:**
1. Open Sonarr web UI -> `Settings` -> `General`
2. Scroll to "API Key" field
3. Copy the API key

**Radarr:**
1. Open Radarr web UI -> `Settings` -> `General`
2. Scroll to "API Key" field
3. Copy the API key

### Add Sonarr to Bazarr:

1. Open Bazarr web UI -> `Settings` -> `Sonarr`
2. Configure the following:
   - Hostname: use `sonarr:8989` if on the same Docker network, or `<host-ip>:8989`
   - Port: `8989` (or your configured Sonarr port)
   - API Key: paste the API key from Sonarr
   - Use SSL: checked only if Sonarr is exposed over HTTPS
   - Automatic Directory: typically `/data/shows` (path where Sonarr stores TV shows)
   - Automatic Download: enable for automatic subtitle download on new episodes
3. Click `Test` to verify Bazarr can contact Sonarr
4. Click `Save` to apply settings

### Add Radarr to Bazarr:

1. Open Bazarr web UI -> `Settings` -> `Radarr`
2. Configure the following:
   - Hostname: use `radarr:7878` if on the same Docker network, or `<host-ip>:7878`
   - Port: `7878` (or your configured Radarr port)
   - API Key: paste the API key from Radarr
   - Use SSL: checked only if Radarr is exposed over HTTPS
   - Automatic Directory: typically `/data/movies` (path where Radarr stores movies)
   - Automatic Download: enable for automatic subtitle download on new movies
3. Click `Test` to verify Bazarr can contact Radarr
4. Click `Save` to apply settings

## 7. Search for and Manage Subtitles

### Manual Subtitle Search

After connecting Sonarr and Radarr, Bazarr will detect your TV shows and movies.

1. Open Bazarr web UI -> `Series` (for TV shows) or `Movies` (for movies)
2. Click on a title to view missing subtitles
3. Select the subtitle language you want
4. Click the search icon to search for subtitles
5. Select a subtitle from the results and click to download

### Automatic Subtitle Download

If enabled in Sonarr/Radarr settings:
- When a new episode/movie is downloaded by Sonarr/Radarr, Bazarr automatically searches for subtitles
- Subtitles are downloaded and matched to the media file automatically
- You can monitor the progress in Bazarr web UI

### Subtitle Settings per Title

Configure subtitle preferences per series/movie:

1. Open a series or movie in Bazarr
2. Click the gear icon to access settings:
   - Languages to download
   - Whether to download forced subtitles
   - Hearing-impaired subtitle preferences
3. Save settings

## 8. Monitor Subtitles and Library

### Subtitle Status Dashboard

Bazarr provides a dashboard showing:
- Missing subtitles per language
- Percentage of library with subtitles
- Recent subtitle downloads
- Failed subtitle downloads

### Subscription Management

Bazarr can notify you of missing subtitles:

1. Open Bazarr web UI -> `Settings` -> `Notifications`
2. Configure notification channels (email, Discord, webhook, etc.)
3. Set which events trigger notifications (missing subtitles, errors, etc.)

### Library Synchronization

Bazarr automatically syncs with Sonarr/Radarr to:
- Detect new episodes/movies
- Update subtitle status
- Monitor file changes

You can manually trigger a sync:
- Bazarr web UI -> `Dashboard` -> `Force library sync` button

## 9. Bazarr Subtitle Management Workflow

```
1. Sonarr/Radarr detects new episode/movie
   ↓
2. Downloads content from NZBGet/qBittorrent
   ↓
3. Post-processes to /data/shows or /data/movies
   ↓
4. Sonarr/Radarr notifies Bazarr of new file
   ↓
5. Bazarr scans the file (title, language, format)
   ↓
6. Searches subtitle providers for matching subtitles
   ↓
7. Downloads and matches subtitles to media file
   ↓
8. Places subtitles alongside media file
   ↓
9. Jellyfin detects subtitles and makes available to user
```

## 10. Troubleshooting

### Bazarr Can't Connect to Sonarr/Radarr
**Symptom:** "Test" fails when adding Sonarr/Radarr connection  
**Solution:**
- Verify both services are running: `docker compose ps sonarr radarr`
- If on same Docker network, use container names: `sonarr:8989` or `radarr:7878`
- If using host IP, verify Bazarr can reach that IP
- Check firewall isn't blocking the ports
- Verify API key is correct (copy from Sonarr/Radarr Settings)

### No Subtitles Found for Files
**Symptom:** Bazarr can't find subtitles for media files  
**Solution:**
- Verify subtitle providers are enabled and configured in Settings -> Subtitles
- Check that languages are configured in Settings -> Languages
- Try manual search with different language selection
- Provider may have rate limiting - wait a few minutes and retry
- File name format may not match database (try manual search)
- Check Bazarr logs for provider error messages

### Automatic Subtitle Download Not Working
**Symptom:** New episodes/movies don't automatically get subtitles  
**Solution:**
- Verify Automatic Download is enabled in Sonarr/Radarr settings in Bazarr
- Check that at least one subtitle provider is configured and active
- Verify languages are configured in Settings -> Languages
- Check Sonarr/Radarr properly notifies Bazarr (use `Force library sync`)
- Review Bazarr logs for errors during search/download
- Try downloading subtitles manually to verify providers work

### Subtitle Download Failures
**Symptom:** Subtitle searches return no results or downloads fail  
**Solution:**
- Provider may have rate limiting - wait before retrying
- Check provider status (some are temporarily down)
- Try different providers if available
- Verify no typos in file names or titles
- Check Bazarr logs for specific provider error messages
- Try manual search to test individual providers

### Bazarr Not Detecting Media Files
**Symptom:** Bazarr shows no series/movies, or list doesn't update  
**Solution:**
- Verify Sonarr/Radarr connection is working (test in settings)
- Click `Force library sync` on dashboard to manually trigger scan
- Check that media files exist in directories Bazarr can access
- Verify `/data/media` volume mount includes correct media paths
- Check Bazarr logs for permission or access errors

### High CPU or Memory Usage
**Symptom:** Bazarr consuming excessive resources  
**Solution:**
- Disable subtitle provider features you don't need
- Review Bazarr logs for continual search loops
- Restart Bazarr: `docker compose restart bazarr`
- If syncing large library, limit library sync frequency in settings
- Check available disk space

### Subtitle Matching Issues
**Symptom:** Downloaded subtitles don't sync with video (out of sync or wrong language)  
**Solution:**
- Different subtitle versions may be needed for different video releases
- Try downloading alternative subtitle sources with manual search
- Ensure encoding is correct (UTF-8 preferred)
- Check media file format matches subtitle format compatibility
- Use manual override to select correct subtitle from multiple options

### Sonarr/Radarr API Key Issues
**Symptom:** "Invalid API key" error in Bazarr  
**Solution:**
- Re-copy API key from Sonarr/Radarr (Settings -> General)
- Paste exactly into Bazarr settings (no extra spaces)
- If API key was regenerated, update in Bazarr immediately
- Test connection again after updating
- Check if API key contains special characters requiring escaping

