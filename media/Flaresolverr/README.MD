Flaresolverr is a reverse proxy that bypasses Cloudflare and Hcaptcha challenges, making it essential for accessing certain torrent indexers through Prowlarr. It works as a supporting service that allows Prowlarr to retrieve data from protected indexers without being blocked by anti-bot protections.

## Directory Structure

Flaresolverr uses a minimal directory structure for configuration:

```
/docker/servarr/flaresolverr
└── /config                    # Flaresolverr configuration (logs and settings)
```

### Create Flaresolverr Directories

Before starting Flaresolverr, create the required directories:

```bash
# Create Flaresolverr config directory
mkdir -p /docker/servarr/flaresolverr

# Set proper permissions (adjust 1000:1000 to your PGID:PUID)
chown -R 1000:1000 /docker/servarr/flaresolverr
chmod 755 /docker/servarr/flaresolverr
```

Or use this one-liner:
```bash
mkdir -p /docker/servarr/flaresolverr && chown -R 1000:1000 /docker/servarr/flaresolverr && chmod 755 /docker/servarr/flaresolverr
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Flaresolverr service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host.

```yaml
flaresolverr:
  container_name: flaresolverr
  environment:
    CAPTCHA_SOLVER: "none"            # Solver for CAPTCHAs (none, chromium, or custom)
    LOG_HTML: "false"                 # Set to true to log HTML responses for debugging
    LOG_LEVEL: "info"                 # Logging level (info, debug, warning)
    TZ: "America/New_York"            # replace with your timezone
  hostname: flaresolverr
  image: ghcr.io/flaresolverr/flaresolverr:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 8191
      published: "8191"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/flaresolverr  # Flaresolverr config and logs
      target: /app/output
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/flaresolverr` with your config storage path.
- Port `8191` is the default Flaresolverr port and should match on both container and host unless you have a specific reason to change it.
- Adjust the environment variables as needed:
  - `LOG_LEVEL`: Set to `info`, `debug`, or `warning` depending on your preferred logging detail
  - `LOG_HTML`: Set to `true` if you want to log HTML responses (useful for debugging)
  - `CAPTCHA_SOLVER`: Can be `none` (no CAPTCHA solving) or `chromium` (for advanced CAPTCHA handling)
  - `TZ`: Set to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`)
- The volume mount stores Flaresolverr logs and output for debugging purposes.
- **Important:** The config directory must exist before Flaresolverr starts to avoid permission issues.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d flaresolverr
```

## 4. Verify

- Check logs: `docker compose logs -f flaresolverr`
- Check service is running: `docker compose ps flaresolverr`
- Verify the service is responding: `curl http://localhost:8191/` (should return JSON response)
- Replace `localhost` with your server IP if not testing locally

## 5. Initial Setup and Configuration

### Verify Flaresolverr Status

After starting the service, verify it's responding correctly:

```bash
# Test the Flaresolverr endpoint
curl http://localhost:8191/

# Should return: {"status":"ok","message":"Flaresolverr is ready!"}
```

### Configure Logging (Optional)

You can adjust Flaresolverr's logging verbosity by modifying the `LOG_LEVEL` environment variable in the compose file:

- `info` (default): Shows general operation messages
- `debug`: Detailed debug information for troubleshooting
- `warning`: Only critical warnings and errors

To enable HTML response logging for debugging complex Cloudflare patterns:

1. Edit your compose file and set `LOG_HTML: "true"`
2. Restart Flaresolverr: `docker compose restart flaresolverr`
3. Logs will be written to `/docker/servarr/flaresolverr/`

## 6. Connect Flaresolverr to Prowlarr

Flaresolverr is used as a proxy solver by Prowlarr to bypass Cloudflare and other anti-bot protections on indexers.

### Prerequisites:
- Prowlarr must be running and reachable from the Flaresolverr host (same Docker network or host IP and port)
- You need the host:port where Flaresolverr is accessible (default: `flaresolverr:8191` on the same compose network or `<host-ip>:8191`)
- Flaresolverr service must be healthy (test with curl as shown above)

### Add Flaresolverr to Prowlarr:

1. Open Prowlarr web UI (`http://<host-ip>:9696`)
2. Navigate to `Settings` -> `Indexers` -> `Options`
3. Locate the `FlareSolverr` section
4. Set the FlareSolverr URL:
   - If Prowlarr and Flaresolverr are on the same Docker network: `http://flaresolverr:8191`
   - If accessing via host IP: `http://<host-ip>:8191` (replace with actual IP)
5. Click `Test` to verify Prowlarr can contact Flaresolverr (should return status: ok)
6. Click `Save` to apply the settings

### Enable Flaresolverr for Protected Indexers:

Once connected to Prowlarr, individual indexers can be configured to use Flaresolverr:

1. In Prowlarr, navigate to `Indexers`
2. Select an indexer that requires Cloudflare/Hcaptcha bypass (commonly torrent indexers)
3. Edit the indexer settings
4. Enable `Use Flaresolverr` toggle (if available for that indexer)
5. Save the indexer configuration

Prowlarr will now automatically route requests through Flaresolverr for protected indexers, allowing it to bypass Cloudflare challenges and retrieve indexer data.

### Common Protected Indexers:

Typical indexers that require Flaresolverr protection on Prowlarr:
- The Pirate Bay (torrents)
- 1337x (torrents)
- Kickass Torrents (torrents)
- Other torrent indexers behind Cloudflare or Hcaptcha

## 7. Flaresolverr Integration Workflow

```
1. User configures indexer in Prowlarr
   ↓
2. Prowlarr detects Cloudflare protection
   ↓
3. Prowlarr sends request to Flaresolverr
   ↓
4. Flaresolverr bypasses Cloudflare/Hcaptcha challenge
   ↓
5. Flaresolverr returns protected content
   ↓
6. Prowlarr processes indexer data
   ↓
7. Available torrent/NZB content shown in Prowlarr
   ↓
8. Sonarr/Radarr can now find content via Prowlarr
```

## 8. Troubleshooting

### Flaresolverr Service Not Responding
**Symptom:** Curl test returns "Connection refused" or timeout  
**Solution:**
- Check service is running: `docker compose ps flaresolverr`
- View logs: `docker compose logs -f flaresolverr`
- Verify port 8191 is exposed: `docker compose port flaresolverr 8191`
- Ensure the service has fully started (wait 10-15 seconds after docker compose up)

### Cloudflare Challenge Still Blocking Prowlarr
**Symptom:** Prowlarr can't retrieve results from Cloudflare-protected indexers  
**Solution:**
- Verify Flaresolverr URL is correct in Prowlarr settings
- Click `Test` in Prowlarr FlareSolverr settings to check connectivity
- Ensure the indexer has `Use Flaresolverr` enabled in its configuration
- Check Flaresolverr logs: `docker compose logs flaresolverr` for error messages
- Cloudflare may have updated protection - Flaresolverr may need updating

### Test Connection from Prowlarr Fails
**Symptom:** Prowlarr shows "Test Failed" when checking Flaresolverr connection  
**Solution:**
- If on same Docker network, verify both services are in `servarrnetwork`
- If using host IP, verify Flaresolverr is accessible from Prowlarr's host
- Check DNS resolution: `docker compose exec prowlarr nslookup flaresolverr`
- Verify firewall isn't blocking port 8191 between services
- Check both services are running: `docker compose ps`

### High CPU or Memory Usage
**Symptom:** Flaresolverr consuming excessive resources  
**Solution:**
- Reduce logging verbosity by setting `LOG_LEVEL: "warning"`
- Set `LOG_HTML: "false"` if it's enabled
- Restart Flaresolverr to clear temporary state: `docker compose restart flaresolverr`
- If running Chromium CAPTCHA solver, verify system has sufficient resources
- Monitor resource usage: `docker stats flaresolverr`

### Flaresolverr Crashes or Restarts
**Symptom:** Docker shows container repeatedly restarting  
**Solution:**
- Check logs for errors: `docker compose logs --tail 50 flaresolverr`
- Common causes: memory limits, invalid environment variables, bad volume mounts
- Verify the config directory exists with proper permissions: `ls -la /docker/servarr/flaresolverr`
- Check available disk space on the host
- Try increasing memory allocation if available

### Indexer Results Still Slow Even with Flaresolverr
**Symptom:** Prowlarr retrieves results but very slowly, or syncing takes a long time  
**Solution:**
- Cloudflare bypass is working but indexer response is slow (not Flaresolverr issue)
- Verify internet connection speed from the host
- Check if indexers are being rate-limited (wait between requests)
- Monitor Flaresolverr resource usage while syncing
- Consider staggering indexer sync intervals in Prowlarr to reduce load

### Chromium CAPTCHA Solver Issues
**Symptom:** Using `CAPTCHA_SOLVER: "chromium"` but challenges still failing  
**Solution:**
- Ensure host has sufficient memory for Chromium process
- Check logs: `docker compose logs flaresolverr | grep -i chromium`
- Verify Docker has permission to run privileged containers if required
- Fallback to `CAPTCHA_SOLVER: "none"` if Chromium is unstable
- Update Flaresolverr image for latest solver improvements
