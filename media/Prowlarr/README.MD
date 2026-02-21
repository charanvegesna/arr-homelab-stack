# Prowlarr Install and Docker-Compose Guide

Prowlarr is an indexer manager and proxy for Sonarr and Radarr. It allows you to manage your torrent and usenet indexers in a central location and provides integration with download clients. Prowlarr also supports Flaresolverr for bypassing Cloudflare and other protection challenges on indexers. All indexers added to Prowlarr are automatically synced to Sonarr and Radarr through API integration.

## Directory Structure

Prowlarr uses the following directory structure (defined in the main media README):

```
/docker/servarr/prowlarr
└── /config                 # Prowlarr configuration, database, and indexer settings
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Prowlarr service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host. This snippet intentionally removes any personal or machine-specific absolute paths.

```yaml
prowlarr:
  cpu_shares: 90
  command: []
  container_name: prowlarr
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"  # replace with your timezone
  hostname: prowlarr
  image: lscr.io/linuxserver/prowlarr:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 9696
      published: "9696"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/prowlarr  # replace with your config path
      target: /config
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/prowlarr` with your config storage path (typically `/docker/servarr/prowlarr`).
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- The config volume stores Prowlarr's database, settings, API keys, and indexer configurations.
- Docker Compose will automatically create all required directories with the `create_host_path: true` setting.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d prowlarr
```

## 4. Verify

- Check logs: `docker compose logs -f prowlarr`
- Visit `http://<host-ip>:9696` to access the Prowlarr web UI (replace `<host-ip>` with the server IP).
- You will be guided through the initial setup wizard on first access.

## 5. Connect Prowlarr to Sonarr and Radarr

Prowlarr acts as a centralized indexer manager and proxy for Sonarr and Radarr, providing unified indexer management instead of configuring each application separately.

### Prerequisites:
- Sonarr and Radarr must be running and reachable from the Prowlarr host (same Docker network or host IP and port).
- You will need the Prowlarr API key and the host:port where it is reachable (Prowlarr default port: `9696`).
- Prowlarr uses Sync Intervals to synchronize indexer lists to Sonarr and Radarr (default: every 12 hours).

### Get the Prowlarr API key:
- Open Prowlarr web UI at `http://<host-ip>:9696`
- Navigate to `Settings` -> `General`
- Copy the `API Key` from the Security section

### Add Prowlarr to Sonarr:

1. Open Sonarr -> `Settings` -> `Indexers`
2. Click the settings icon (gear icon) in the top right corner or look for `Prowlarr` options
3. Navigate to the `Prowlarr` settings tab (or search for Prowlarr)
4. Configure the following:
   - Enable: `checked`
   - Prowlarr Server URL: the address where Prowlarr is reachable
     - If on the same Docker network: `http://prowlarr:9696`
     - If using host IP: `http://<host-ip>:9696`
   - API Key: paste the Prowlarr API key from above
   - Sync Interval: `12` hours (default, adjust as needed for more frequent syncs)
   - Additional Options: keep defaults or adjust priority if needed
5. Click `Test` to verify Sonarr can successfully contact Prowlarr
6. Click `Save` to add Prowlarr

Once configured, Sonarr will sync all enabled indexers from Prowlarr every 12 hours (or your configured interval) and use them to search for series.

### Add Prowlarr to Radarr:

1. Open Radarr -> `Settings` -> `Indexers`
2. Click the settings icon (gear icon) in the top right corner or look for `Prowlarr` options
3. Navigate to the `Prowlarr` settings tab (or search for Prowlarr)
4. Configure the following:
   - Enable: `checked`
   - Prowlarr Server URL: the address where Prowlarr is reachable
     - If on the same Docker network: `http://prowlarr:9696`
     - If using host IP: `http://<host-ip>:9696`
   - API Key: paste the Prowlarr API key from above
   - Sync Interval: `12` hours (default, adjust as needed for more frequent syncs)
   - Additional Options: keep defaults or adjust priority if needed
5. Click `Test` to verify Radarr can successfully contact Prowlarr
6. Click `Save` to add Prowlarr

Once configured, Radarr will sync all enabled indexers from Prowlarr every 12 hours (or your configured interval) and use them to search for movies.

## 6. Add Indexers to Prowlarr

After connecting Prowlarr to Sonarr and Radarr, add indexers that will be used for searching:

1. Open Prowlarr web UI -> `Indexers` tab
2. Click `Add Indexer` button or the `+` icon
3. Search for and select your desired indexers:
   - **Torrent indexers** (e.g., The Pirate Bay, 1337x, RARBG, etc.)
   - **Usenet indexers** (e.g., NZBgeek, NZB.su, Drunkenslug, etc.)
   - **Public indexers** (e.g., OMDB, YTS, etc.)
4. For each indexer, configure:
   - **Base URL**: Usually pre-filled by Prowlarr
   - **API Key**: Enter if required by the indexer (check indexer's website or Prowlarr's suggestions)
   - **Proxy**: Set to your VPN or proxy service if needed (optional but recommended for privacy)
   - **Priority**: Set priority for multi-indexer searches (lower number = higher priority; default: `25`)
   - **Categories**: Select which content types the indexer provides (Movies, TV Shows, Music, etc.)
   - **Enable**: Check to activate the indexer for searches
5. Click `Test` to verify the indexer is accessible and working
6. Click `Save` to add the indexer

Once an indexer is added and enabled, it will automatically sync to Sonarr and Radarr within the configured sync interval.

### Indexer Suggestions:
- **Torrent Indexers**: The Pirate Bay, 1337x, RARBGProxyBot, Public indexers
- **Usenet Indexers**: NZBgeek, NZB.su, Drunkenslug (requires API key from usenet provider)
- **Quality**: Configure indexer download link sources and quality preferences within each indexer's settings

## 7. Configure Flaresolverr (Optional)

If you have Flaresolverr running and need to bypass Cloudflare and similar protection services on certain indexers:

### Prerequisites for Flaresolverr Integration:
- Flaresolverr must be running and reachable from the Prowlarr container
- Flaresolverr default port: `8191`
- Some indexers may require Flaresolverr bypass, while others may work without it

### Setup Prowlarr Flaresolverr Integration:

1. Open Prowlarr -> `Settings` -> `Indexers` -> `Options`
2. Find the `FlareSolverr` settings section
3. Set the FlareSolverr URL:
   - If on the same Docker network: `http://flaresolverr:8191`
   - If using host IP: `http://<host-ip>:8191`
4. Click `Test` to verify Prowlarr can successfully contact Flaresolverr
5. Save the settings

### Enable Flaresolverr for Specific Indexers:

1. Open Prowlarr -> `Indexers`
2. Click `Edit` on an indexer that requires Cloudflare bypass
3. Look for the `Use Flaresolverr` or `Bypass` option
4. Check/Enable: `Use Flaresolverr` (if available for that indexer)
5. Click `Save`

Once configured, Prowlarr will automatically route requests through Flaresolverr for indexers that need Cloudflare bypassing. These requests will be passed to Sonarr and Radarr, allowing searches on protected indexers.

### Notes:
- Flaresolverr adds latency to searches; only enable it for indexers that require it
- Monitor Flaresolverr logs if searches against protected indexers fail
- Some indexers may need Flaresolverr enabled even if they don't appear to have Cloudflare protection

## 8. Index Sync and Search Workflow

```
1. Indexers configured in Prowlarr
   ↓
2. Sync Interval triggers (every 12 hours by default)
   ↓
3. Prowlarr sends indexer list to Sonarr and Radarr
   ↓
4. User searches for Series/Movie in Sonarr/Radarr
   ↓
5. Request sent to Prowlarr with search query
   ↓
6. Prowlarr queries all enabled indexers in parallel
   ↓
7. Results from indexers collected (Flaresolverr used if needed)
   ↓
8. Results returned to Sonarr/Radarr for user review
   ↓
9. User selects result and sends to download client
   ↓
10. Download client (NZBGet, qBittorrent, etc.) processes the download
```

## Troubleshooting

### Prowlarr cannot connect to Sonarr/Radarr:
- Verify network connectivity: Both services must be on the same Docker network or reachable via host IP
- Check API keys in both applications
- Confirm port numbers (Prowlarr: 9696, Sonarr: 8989, Radarr: 7878)
- Check firewall rules if using host IP connection

### Indexers not syncing to Sonarr/Radarr:
- Verify Prowlarr is enabled in Sonarr/Radarr settings
- Check sync interval and trigger a manual sync if available
- Ensure at least one indexer is enabled in Prowlarr
- Check Prowlarr logs for sync errors

### Search results not appearing:
- Verify indexers are enabled in Prowlarr
- Test each indexer individually in Prowlarr
- Check if Flaresolverr is needed for that indexer
- Verify API keys if the indexer requires authentication
- Check Prowlarr and indexer logs for error messages
