# Jellyseerr Install and Docker-Compose Guide

Jellyseerr is a free and open-source software application for managing requests for your media library. It integrates with Jellyfin, Sonarr, and Radarr to allow users to discover and request movies and TV shows that are added to your collection automatically. It provides a user-friendly interface for media discovery, request management, and approval workflows.

## Directory Structure

Jellyseerr uses the following directory structure (defined in the main media README):

```
/docker/servarr/jellyseerr
└── /config                 # Jellyseerr configuration, database, and settings
```

### Create Jellyseerr Directories

Before starting Jellyseerr, create the required directories:

```bash
# Create Jellyseerr config directory
mkdir -p /docker/servarr/jellyseerr

# Set proper permissions (adjust 1000:1000 to your PGID:PUID)
chown -R 1000:1000 /docker/servarr/jellyseerr
find /docker/servarr/jellyseerr -type d -exec chmod 755 {} \;
```

Or use this one-liner:
```bash
mkdir -p /docker/servarr/jellyseerr && chown -R 1000:1000 /docker/servarr/jellyseerr
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Jellyseerr service to your compose file

- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host. This snippet intentionally removes any personal or machine-specific absolute paths.

```yaml
jellyseerr:
  cpu_shares: 90
  command: []
  container_name: jellyseerr
  deploy:
    resources:
      limits:
        memory: "4004511744"
  environment:
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"  # replace with your timezone
  hostname: jellyseerr
  image: fallenbagel/jellyseerr:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 5055
      published: "5055"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/jellyseerr  # replace with your config path
      target: /app/config
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/jellyseerr` with your config storage path (typically `/docker/servarr/jellyseerr`).
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- The memory limit is set to ~4GB; adjust if needed based on your system resources.
- The config volume stores Jellyseerr's settings, database, user accounts, and request history.
- **Important:** The config directory must exist before Jellyseerr starts to avoid permission issues.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d jellyseerr
```

## 4. Verify

- Check logs: `docker compose logs -f jellyseerr`
- Visit `http://<host-ip>:5055` to access the Jellyseerr web UI (replace `<host-ip>` with the server IP).
- You will be guided through the initial setup wizard on first access.

## 5. Initial Setup and Configuration

When you first access the Jellyseerr web UI, perform these essential configuration steps:

1. Open `http://<host-ip>:5055` in your browser
2. You will be guided through the initial setup wizard on first access
3. After wizard completes, configure additional settings:

### Connect to Jellyfin:

Jellyfin provides the media library interface for users:

1. Open Jellyseerr -> `Settings` -> `Jellyfin`
2. Configure the following:
   - **Host**: the address where Jellyfin is reachable
     - If on the same Docker network: `http://jellyfin:8096`
     - If using host IP: `http://<host-ip>:8096`
   - **Username**: your Jellyfin admin username
   - **Password**: your Jellyfin admin password
   - **Library Sync**: Enable `checked` (automatically sync libraries)
3. Click `Connect` to verify connection
4. Click `Save` to store settings

Once connected, Jellyseerr will sync your Jellyfin library and display available media to users.

### Connect to Sonarr:

Sonarr handles automatic TV show requests:

1. Open Jellyseerr -> `Settings` -> `Sonarr`
2. Configure the following:
   - **Host**: the address where Sonarr is reachable
     - If on the same Docker network: `sonarr:8989`
     - If using host IP: `<host-ip>:8989`
   - **Port**: `8989`
   - **API Key**: found in Sonarr -> `Settings` -> `General` -> `Security` -> `API Key`
   - **URL Base**: Leave empty unless Sonarr is behind a reverse proxy
   - **Root Folder**: Select the root folder where TV shows are stored (usually `/shows`)
   - **Quality Profile**: select your preferred quality profile for TV shows
   - **Language Profile**: select language preferences if applicable
   - **Enable Season Folders**: `checked` (recommended)
   - **Use SSL**: `unchecked` (unless configured)
3. Click `Test` to verify the connection
4. Click `Save` to store settings

Once configured, TV show requests will be automatically added to Sonarr.

### Connect to Radarr:

Radarr handles automatic movie requests:

1. Open Jellyseerr -> `Settings` -> `Radarr`
2. Configure the following:
   - **Host**: the address where Radarr is reachable
     - If on the same Docker network: `radarr:7878`
     - If using host IP: `<host-ip>:7878`
   - **Port**: `7878`
   - **API Key**: found in Radarr -> `Settings` -> `General` -> `Security` -> `API Key`
   - **URL Base**: Leave empty unless Radarr is behind a reverse proxy
   - **Root Folder**: Select the root folder where movies are stored (usually `/movies`)
   - **Quality Profile**: select your preferred quality profile for movies
   - **Minimum Availability**: Set when movie is considered available (default: Released)
   - **Use SSL**: `unchecked` (unless configured)
3. Click `Test` to verify the connection
4. Click `Save` to store settings

Once configured, movie requests will be automatically added to Radarr.

## 6. Create and Manage Users

Jellyseerr allows multiple users to request content:

### Create User Accounts:

1. Open Jellyseerr -> `Settings` -> `Users`
2. Click `Add User` or invite button
3. Configure the following:
   - **Email**: User's email address
   - **Username**: Unique username for login
   - **Password**: Initial password (user can change later)
   - **Permissions**:
     - **Is Admin**: Check to give full admin access
     - **Request TV/Movies**: Allow user to request content
     - **Approve Requests**: Allow user to approve requests from others
     - **Manage Issues**: Allow user to manage media issues/reports
4. Click `Create` to add the user

Users can then log in with their credentials and request content.

### User Roles:
- **Admin**: Full access to all settings and user management
- **Regular User**: Can request content and view library (if approved)
- **Guest**: Read-only access to library

## 7. Configure Request Settings

Control how requests are handled:

1. Open Jellyseerr -> `Settings` -> `Request`
2. Configure the following:
   - **Auto-Approve**: Automatically approve requests (or require admin approval)
   - **New User Requests**: Limit requests from new users
   - **Enable Request Notifications**: Send notifications when requests are made/approved
   - **TV Request Quantity**: Maximum episodes user can request at once
   - **Movie Request Quantity**: Maximum movies user can request at once
3. Click `Save` to apply settings

## 8. Configure Notifications

Set up alerts for requests and status updates:

1. Open Jellyseerr -> `Settings` -> `Notifications`
2. Available notification channels:
   - **Email**: Send notifications via email
   - **Discord**: Send notifications to Discord servers
   - **Slack**: Send notifications to Slack channels
   - **Telegram**: Send notifications via Telegram
   - **Webhook**: Custom webhooks for integrations
3. Configure desired channels with appropriate credentials/tokens
4. Click `Test` to verify each notification channel
5. Click `Save` to apply settings

Notifications will be sent when:
- Requests are made by users
- Requests are approved/denied
- Content becomes available
- Issues are reported

## 9. Content Discovery and Libraries

### Browse Content:

1. **Home Tab**: Shows recently added content and recommendations
2. **Discover Tab**: Browse movies and TV shows by category/genre
3. **Movies**: Filter and search for available movies
4. **TV Shows**: Filter and search for available TV shows
5. **Requests**: View pending, approved, and completed requests (admin view)

### Request Content:

1. Browse or search for desired movie/show
2. Click the title to view details
3. Click `Request` button
4. For TV shows, select which seasons to request
5. Submit request
6. If auto-approve enabled, it will be sent to Sonarr/Radarr immediately
7. If approval required, admin will review and approve
8. Once approved, content will be added to your media server

## 10. Monitor Requests and Library

### Dashboard:
- Check main page for overview of requests and library stats
- View recent activity and request status

### Request Management (Admin):
1. Open Jellyseerr -> `Requests`
2. View pending requests from all users
3. Sort by date, user, content type, or status
4. Click request to view details
5. Approve, deny, or modify requests as needed
6. Monitor which requests are fulfilled vs. still downloading

### Library Integration:
- Movies and shows added by Jellyseerr appear in Jellyfin automatically
- Users can see when their requests become available
- Content organized by Sonarr/Radarr naming conventions

### Regular Maintenance:
- Monitor Jellyseerr performance: `docker-compose logs -f jellyseerr`
- Update container regularly: `docker-compose pull && docker-compose up -d jellyseerr`
- Review user activity and request patterns
- Adjust quality profiles and limits as needed
- Monitor disk space for incoming downloads

## 11. Request Workflow

```
1. User discovers content in Jellyseerr
   ↓
2. User clicks Request button
   ↓
3. Request submitted (TV show: select seasons/episodes; Movie: select quality)
   ↓
4. Auto-approve enabled? -> Automatically sent to Sonarr/Radarr
   OR Approval required? -> Admin reviews request
   ↓
5. Admin approves request / OR auto-added
   ↓
6. Sonarr/Radarr receives request
   ↓
7. Search indexers for matching content
   ↓
8. Send to download client (qBittorrent/NZBGet)
   ↓
9. Download completes
   ↓
10. Post-processing and organization
   ↓
11. Content added to /data/shows or /data/movies
   ↓
12. Jellyfin library updated
   ↓
13. Content available in Jellyfin for streaming
   ↓
14. User notified (if configured)
```

## 12. Troubleshooting

### Cannot connect to Jellyfin:
- Verify Jellyfin URL is correct (use `http://jellyfin:8096` if on same network)
- Check credentials (admin username/password)
- Ensure firewall allows port 8096
- Verify Jellyfin is running: `docker-compose logs -f jellyfin`
- Test connection button in Jellyseerr settings

### Cannot connect to Sonarr/Radarr:
- Verify host and port are correct
- Check API key in both Jellyseerr and Sonarr/Radarr settings
- Ensure services are running: `docker-compose logs -f sonarr` or `radarr`
- Verify networks are connected (same Docker network or host IP)
- Click Test button in Jellyseerr settings

### Requests not appearing in Sonarr/Radarr:
- Verify connection is successful (green checkmark in settings)
- Check that root folder is selected in Jellyseerr settings
- Verify quality profile is selected
- Check Jellyseerr logs for error messages: `docker-compose logs -f jellyseerr`
- Ensure Sonarr/Radarr download clients are configured

### Users cannot log in:
- Verify user account exists in Jellyseerr -> `Settings` -> `Users`
- Check that user account is enabled
- Reset password if needed (admin function)
- Check Jellyseerr logs for authentication errors

### Requests not being auto-approved:
- Verify auto-approve is enabled in `Settings` -> `Request`
- Check that Sonarr/Radarr connections are successful
- Verify user has permission to request content
- Check logs for any errors: `docker-compose logs -f jellyseerr`

### Notifications not sending:
- Verify notification channel is configured in `Settings` -> `Notifications`
- Click Test button to verify channel connectivity
- Check that notification settings have correct API keys/tokens
- Ensure email/service provider settings are correct
- Review Jellyseerr logs for notification errors

### Library appearing empty:
- Refresh Jellyfin library sync: `Settings` -> `Jellyfin` -> click refresh button
- Verify Jellyfin connection is successful
- Check Jellyfin is properly indexed in database
- Verify permissions allow Jellyseerr to see libraries
- Monitor Jellyseerr logs for sync errors

### High memory usage:
- Reduce cache size in settings if available
- Monitor user activity and reduce concurrent users if needed
- Increase memory limit in compose.yaml if required
- Restart Jellyseerr: `docker-compose restart jellyseerr`
