# Jellyfin Install and Docker-Compose Guide

Jellyfin is a free and open-source media server and streaming platform. It organizes, plays, and streams your personal media collection (movies, TV shows, music, photos) to users on your network or over the internet. Jellyfin is the central hub of your media stack, displaying all content managed by Sonarr, Radarr, and other backend services.

## Directory Structure

Jellyfin uses the following directory structure:

```
/docker/servarr/jellyfin
└── /config                    # Jellyfin configuration, database, cache, and metadata

/data/media
├── /shows                     # TV show library (mounted as read-only for playback)
├── /movies                    # Movie library (mounted as read-only for playback)
└── /music                     # Music library (optional, mounted as read-only)
```

### Create Jellyfin Directories

Before starting Jellyfin, create the required directories:

```bash
# Create Jellyfin config directory
mkdir -p /docker/servarr/jellyfin

# Set proper permissions (adjust 1000:1000 to your PGID:PUID)
chown -R 1000:1000 /docker/servarr/jellyfin
chmod 755 /docker/servarr/jellyfin
```

Or use this one-liner:
```bash
mkdir -p /docker/servarr/jellyfin && chown -R 1000:1000 /docker/servarr/jellyfin && chmod 755 /docker/servarr/jellyfin
```

## 1. Install Docker
- Follow the instructions in the `infrastructure` directory to install Docker and Docker Compose (see `infrastructure/docker/README.MD`).

## 2. Add Jellyfin service to your compose file
- Change to the compose folder for Servarr services and open the compose file for editing:

```bash
cd /docker/servarr
nano compose.yaml
```

- Add the following sanitized service block to the compose file. Replace the placeholder host paths and variables with values appropriate for your host.

```yaml
jellyfin:
  container_name: jellyfin
  environment:
    JELLYFIN_PublishedServerUrl: "http://localhost:8096"  # URL where Jellyfin is accessible
    PGID: "1000"
    PUID: "1000"
    TZ: "America/New_York"            # replace with your timezone
  hostname: jellyfin
  image: jellyfin/jellyfin:latest
  networks:
    servarrnetwork: null
  ports:
    - mode: ingress
      target: 8096
      published: "8096"
      protocol: tcp
  restart: unless-stopped
  volumes:
    - type: bind
      source: /docker/servarr/jellyfin  # Jellyfin config and database
      target: /config
      bind:
        create_host_path: true
    - type: bind
      source: /data/media/shows        # TV shows library
      target: /shows
      read_only: true
      bind:
        create_host_path: true
    - type: bind
      source: /data/media/movies       # Movies library
      target: /movies
      read_only: true
      bind:
        create_host_path: true
```

## Notes:
- Replace `/docker/servarr/jellyfin` with your config storage path (typically `/docker/servarr/jellyfin`).
- Replace `http://localhost:8096` with the URL where Jellyfin will be accessible (important for remote access and API).
- Adjust `PGID`/`PUID` to match the user that should own the files on the host.
- Set `TZ` to your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Kolkata`).
- Media volumes are mounted as read-only to prevent accidental modifications from Jellyfin.
- The `/shows` and `/movies` mounts are best kept separate from `/config` for performance and organization.
- You can add additional volumes for `/music` or `/photos` if needed: same pattern with `target: /music` and `source: /data/media/music`.
- **Important:** All directories must exist before Jellyfin starts to avoid permission issues.

## 3. Start the service

```bash
# from the folder containing the compose file
docker compose up -d jellyfin
```

## 4. Verify

- Check logs: `docker compose logs -f jellyfin`
- Check service is running: `docker compose ps jellyfin`
- Visit `http://localhost:8096` to access Jellyfin web UI (replace `localhost` with your server IP if testing remotely)
- Initial setup wizard should launch on first access

## 5. Initial Setup and Configuration

### First Access - Jellyfin Setup Wizard

When you first access Jellyfin, a setup wizard will guide you through initial configuration:

1. **Preferred Display Language:** Select your display language (English, Spanish, French, etc.)
2. **Allow Remote Connections:** Choose whether to allow access from outside your local network (disables auto-discovery)
3. **Configure User Account:** Create your admin user account
4. **Set Library Folders:** Add folders where your media is located

### Add Media Libraries

Libraries organize your content by type and make it discoverable in Jellyfin:

1. Open Jellyfin web UI -> `Administration` -> `Libraries`
2. Click `+` to add a new library
3. Configure the library:
   - **Library Name:** e.g., "TV Shows", "Movies"
   - **Content Type:** Select the folder type (TV Series, Movies, Music, etc.)
   - **Folders:** Click `+` to add the folder path (e.g., `/shows` for TV, `/movies` for movies)
   - **Advanced settings:** Configure metadata fetchers, image preferences, scanning behavior
4. Click `Create Library` to add it

### Configure Metadata Providers

Jellyfin automatically fetches metadata (posters, descriptions, ratings, cast info) from external providers:

1. Open Jellyfin web UI -> `Administration` -> `Metadata`
2. Configure providers per content type:
   - **Movie libraries:** TMDb (The Movie Database) - provides posters, descriptions, ratings
   - **TV libraries:** TheTVDB or TVMaze - provides episode data, posters, descriptions
   - **Order of providers:** Drag to set preference order (first match wins)
3. Configure per-provider settings (language, image resolution, etc.)
4. Save settings

### Configure Playback Settings

Customize how media is transcoded and streamed:

1. Open Jellyfin web UI -> `Administration` -> `Playback`
2. Configure transcoding (CPU usage):
   - **Video Codec:** Select transcoding options (H.264 preferred for compatibility)
   - **Bitrate Limits:** Set max streaming bitrate for remote users
   - **VAAPI/NVENC:** Enable hardware acceleration if your system supports it
3. Configure subtitle options:
   - **Subtitle Mode:** Automatic, Always, Default font
   - **Burn Subtitles:** Encode subtitles into video (more CPU-intensive)
4. Save settings

## 6. Library Management

### Library Scanning and Refresh

Jellyfin automatically detects new media when files are added:

**Automatic Scanning:**
- Enabled by default - watches library folders for new files
- Detects new episodes added by Sonarr, movies by Radarr within ~5-10 minutes
- Automatic metadata fetching starts after detection

**Manual Library Refresh:**
1. Open Jellyfin web UI -> `Administration` -> `Scheduled Tasks`
2. Run "Scan Library" immediately (don't wait for automatic schedule)
3. Or within Library view, click refresh icon on library name

### Organizing Collections

Create custom collections to organize media across your library:

1. Open Jellyfin web UI and navigate to a library
2. Click the collection icon (usually in header)
3. Create a new collection or add items to existing ones
4. Collections appear in the UI for easy browsing

### Metadata Editing

Manually edit incorrect metadata for media:

1. Open a movie or TV show detail page
2. Click the edit icon (pencil)
3. Modify:
   - Title, year, runtime
   - Poster image, backdrop
   - Description, genres, cast
   - Episode information (for TV shows)
4. Click save to apply changes

## 7. User Management

### Create User Accounts

Users can log in and access your Jellyfin library:

1. Open Jellyfin web UI -> `Administration` -> `Users`
2. Click `+` to add a new user
3. Configure the user:
   - **Username:** Login name
   - **Password:** User's access password
   - **Access permissions:** Select which libraries the user can access
   - **User policies:** Set watch time limits, content restrictions, etc.
4. Click `Create` to add the user

### User Access Control

Restrict what content each user can access:

1. Open user settings in `Administration` -> `Users` -> (select user)
2. **Library Access:** Select which libraries the user can browse
3. **Item Access:** Optionally restrict access by content rating
4. **Features:** Enable/disable downloading, sync, server administration
5. Save changes

### Shared Access and Invitations

For remote users on your Jellyfin server:

1. Create a user account for each person
2. Provide them with:
   - Server URL: `http://<your-server-ip>:8096`
   - Username and password
   - Library recommendations
3. Users log in from their devices (web, mobile apps, TV apps, etc.)

## 8. Playback and Streaming

### Direct Stream vs Transcode

Jellyfin handles different playback scenarios:

**Direct Stream (Preferred):**
- Server sends file as-is (no re-encoding)
- Low CPU usage
- Requires compatible player and network bandwidth

**Transcode:**
- Server converts video/audio format on-the-fly
- Allows older devices to play new formats
- High CPU usage (consider hardware acceleration)
- Requires sufficient bandwidth

### Supported Playback Devices

**Web Browsers:** Chrome, Firefox, Safari, Edge
- Full feature support
- Configure in `User Preferences` -> `Playback` on first login

**Mobile Apps:** iOS (official app), Android (official + third-party)
- Download from App Store/Play Store
- Allows offline downloads of content

**TV Apps:** Roku, Apple TV, Android TV, Samsung TV, LG TV
- Download respective apps from app stores
- Seamless UX for living room viewing

**DLNA Devices:** Chromecast, AirPlay, DLNA receivers
- Via Jellyfin app or browser
- Stream to any DLNA-compatible device

### Remote Access

Access Jellyfin outside your local network:

**Via Public URL (if configured):**
1. Server must be accessible from the internet
2. Jellyfin auto-detects public URL
3. Users access via `https://<your-domain>:8096`

**Via VPN:**
1. Connect to VPN first from remote location
2. Access Jellyfin on local network as usual
3. Most secure option

**Via Reverse Proxy:**
1. Set up nginx or Caddy reverse proxy
2. Maps external domain to Jellyfin port
3. Allows HTTPS without exposing Jellyfin directly

## 9. Jellyfin Media Streaming Workflow

```
1. Sonarr/Radarr downloads new content
   ↓
2. Post-processes to /data/shows or /data/movies
   ↓
3. Jellyfin auto-detects new files
   ↓
4. Fetches metadata from providers (TMDb, TheTVDB)
   ↓
5. Downloads posters, descriptions, ratings
   ↓
6. Creates library entries for browsing
   ↓
7. Indexes content for search functionality
   ↓
8. Users browse library and select content
   ↓
9. Jellyfin determines optimal playback format
   ↓
10. Streams directly or transcodes if needed
   ↓
11. User watches on their device (web, mobile, TV)
```

## 10. Integration with Jellyseerr

Jellyseerr provides a user-friendly interface for requesting content:

### Prerequisites:
- Jellyfin must be running and accessible
- Sonarr and Radarr must be configured in Jellyseerr
- Users must have Jellyfin accounts to request content

### Basic Integration:
1. Jellyseerr connects to your Jellyfin server
2. Users browse movies/shows in Jellyseerr
3. Click "Request" to add content to Sonarr/Radarr
4. Content is downloaded and added to Jellyfin library automatically
5. User is notified when content is available

### Notifications When Content Available:
- Jellyseerr can notify users when requested content is available
- Jellyfin displays new content in "Continue Watching" or library
- Users see newly added episodes/movies when they request them

## 11. Troubleshooting

### Jellyfin Can't Connect to Network
**Symptom:** Can't access `http://localhost:8096` or server not found  
**Solution:**
- Check service is running: `docker compose ps jellyfin`
- Verify port mapping: `docker compose port jellyfin 8096`
- Check logs: `docker compose logs jellyfin`
- Verify firewall isn't blocking port 8096
- If using host IP, verify Jellyfin running on correct interface (not localhost-only)

### Media Files Not Appearing in Library
**Symptom:** Added files to media folder but Jellyfin doesn't detect them  
**Solution:**
- Check file is in correct library folder (e.g., `/data/media/shows` for TV shows)
- Check file format is supported (MKV, MP4, AVI, MOV commonly supported)
- Manually trigger library refresh in `Administration` -> `Scheduled Tasks`
- Check Jellyfin logs for permission errors accessing media files
- Verify media folder volume mount is correct in compose file
- Wait 5-10 minutes for automatic scan to complete

### Metadata Not Fetching (Blank Posters/Descriptions)
**Symptom:** Media appears with no poster, description, or wrong data  
**Solution:**
- Check metadata providers are configured: `Administration` -> `Metadata`
- Verify provider order (first match is used)
- Manually refresh metadata: click edit on media item and re-fetch
- Some content may not exist in metadata databases (try manual edit)
- Check provider limits/rate limiting (TMDb has free API limits)
- Try alternate providers if one isn't working

### Video Playback Issues (Buffering, Stuttering)
**Symptom:** Video stutters, buffers frequently, or stops during playback  
**Solution:**
- Check network bandwidth - verify server connection speed
- Reduce stream quality in playback settings
- Check if transcoding is needed (see Playback Settings)
- Enable hardware acceleration if available (`Administration` -> `Playback`)
- Check server CPU/memory usage (system load too high)
- Move media to faster storage if possible

### Can't Access Jellyfin Remotely
**Symptom:** Can access locally but not from outside network  
**Solution:**
- Verify server address in `JELLYFIN_PublishedServerUrl` matches external access URL
- Check port forwarding: ensure port 8096 is forwarded to server
- Check firewall: port 8096 must be open for remote access
- For security, consider using reverse proxy with HTTPS
- Verify no double-NAT (ISP NAT + router NAT can block access)

### Users Can't Log In or Permission Errors
**Symptom:** Cannot log in as user or permission denied accessing libraries  
**Solution:**
- Verify user account exists: `Administration` -> `Users`
- Check library access permissions for user
- Verify user policy doesn't have access restricted
- For new users, ensure they have at least one library assigned
- Check password is correct (case-sensitive)
- Clear browser cache and cookies, then retry login

### Subtitle Sync Issues or Not Displaying
**Symptom:** Subtitles missing, not syncing, or displaying incorrectly  
**Solution:**
- Verify subtitles are in same folder as video with same filename
- Ensure subtitle file format is supported (SRT, ASS, VOBSUB, etc.)
- Check playback settings: `Administration` -> `Playback` for subtitle options
- Enable "Always" subtitle mode to force display
- Try different subtitle encoding (UTF-8 preferred)
- In Bazarr, verify subtitles are downloading to media folder correctly

### High CPU/Memory Usage
**Symptom:** Jellyfin consuming excessive CPU or memory  
**Solution:**
- Check if transcoding is occurring (disable if unnecessary)
- Reduce concurrent user limit in `Administration` -> `Server`
- Enable hardware acceleration for transcoding
- Optimize metadata fetching - limit number of metadata providers
- Reduce library scan frequency (don't scan too often)
- Check for library corruption: rebuild library index in scheduled tasks

### WebUI Very Slow or Lag
**Symptom:** Web interface responds slowly or lags when browsing  
**Solution:**
- Check network connection speed
- Clear browser cache
- Try another browser to isolate client vs server issue
- Check if library is very large (optimize with collections)
- Check server resource usage (CPU, memory, disk I/O)
- Reduce number of items shown per page in display settings
- Restart Jellyfin: `docker compose restart jellyfin`

### SSL/HTTPS Not Working
**Symptom:** HTTPS not working or certificate errors when accessing Jellyfin  
**Solution:**
- Jellyfin by default uses HTTP only
- For HTTPS, use a reverse proxy (nginx, Caddy) in front
- Install SSL certificate on reverse proxy, not Jellyfin
- Update `JELLYFIN_PublishedServerUrl` to use `https://` if reverse proxy configured
- Self-signed certs will show warnings - use Let's Encrypt for valid cert
- Verify reverse proxy is properly forwarding HTTPS to Jellyfin on port 8096
