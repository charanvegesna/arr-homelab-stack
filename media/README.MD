# Media Server Stack - Directory Structure and Setup Guide

## Directory Tree Structure

```
/data (media storage root)
├── /movies                          # Movie library
├── /shows                           # TV shows library
├── /music                           # Music library
└── /downloads                       # Download staging area
    ├── /nzbget
    │   ├── /completed               # Completed usenet downloads
    │   └── /intermediate            # In-progress usenet downloads
    └── /qbittorrent
        ├── /completed               # Completed torrent downloads
        ├── /incomplete              # In-progress torrent downloads
        └── /torrents
            └── /intermediate        # Temporary torrent files

/docker (docker configuration root)
├── /jellyfin
│   └── compose.yaml                 # Jellyfin container config
└── /servarr
    └── compose.yaml                 # Servarr services config (Sonarr, Radarr, etc.)
```

## Directory Creation Commands

### Create Media Storage Structure

```bash
# Create root data directory
mkdir -p /data/movies
mkdir -p /data/shows
mkdir -p /data/music

# Create download directories for NZBGet
mkdir -p /data/downloads/nzbget/completed
mkdir -p /data/downloads/nzbget/intermediate

# Create download directories for qBittorrent
mkdir -p /data/downloads/qbittorrent/completed
mkdir -p /data/downloads/qbittorrent/incomplete
mkdir -p /data/downloads/qbittorrent/torrents/intermediate

# One-liner to create all data directories
mkdir -p /data/{movies,shows,music,downloads/{nzbget/{completed,intermediate},qbittorrent/{completed,incomplete,torrents/intermediate}}}
```

### Create Docker Configuration Structure

```bash
# Create docker configuration directories
mkdir -p /docker/jellyfin
mkdir -p /docker/servarr

# Create servarr service config directories (for persistent storage)
mkdir -p /docker/servarr/{sonarr,radarr,prowlarr,bazarr,jellyseerr,qbittorrent,nzbget}

# One-liner for docker structure
mkdir -p /docker/{jellyfin,servarr/{sonarr,radarr,prowlarr,bazarr,jellyseerr,qbittorrent,nzbget}}
```

### Set Permissions (Linux/Unix)

```bash
# Set ownership for user:group (adjust 1000:1000 to match your PGID:PUID)
chown -R 1000:1000 /data
chown -R 1000:1000 /docker

# Set permissions (755 for directories, 644 for files)
find /data -type d -exec chmod 755 {} \;
find /docker -type d -exec chmod 755 {} \;
```

## Installation Order and Why

### 1. **First: Create Directory Structure** (No services needed yet)
   - Ensure all paths exist before containers start
   - Prevents `create_host_path: true` from creating wrong directories
   - Allows you to pre-set correct permissions

### 2. **Second: Install Jellyfin** (Media Library Server)
   - **Why first?** Jellyfin is the **foundation** - it serves your media library to users
   - Other services depend on Jellyfin existing for Jellyseerr to connect
   - Independent of download services (Sonarr/Radarr)
   - Run in separate compose file: `/docker/jellyfin/compose.yaml`

### 3. **Third: Install Servarr Services** (Download/Management Stack)
   - **Why after Jellyfin?** This stack manages downloads and adds content to both Jellyfin and Jellyseerr
   - Installation order within servarr:
     1. **Prowlarr** - Indexer manager (provides search sources)
     2. **Sonarr** - TV show manager (uses Prowlarr for searches)
     3. **Radarr** - Movie manager (uses Prowlarr for searches)
     4. **Bazarr** - Subtitle manager (connects to both Sonarr and Radarr)
     5. **qBittorrent** - Torrent downloader (downloads from Sonarr/Radarr)
     6. **NZBGet** - Usenet downloader (downloads from Sonarr/Radarr)
     7. **Jellyseerr** - Request manager (users request, Sonarr/Radarr fulfill)
   - Run in shared compose file: `/docker/servarr/compose.yaml`

## Why Separate Jellyfin and Servarr Directories

### Benefits of Separation:

1. **Service Independence**
   - Jellyfin manages and serves media
   - Servarr services manage downloads and organization
   - Can restart one stack without affecting the other

2. **Scalability**
   - Jellyfin can run on different hardware (high streaming demand)
   - Servarr services require less resources, can run together
   - Easy to migrate Jellyfin to dedicated server later

3. **Security**
   - Jellyfin is user-facing (exposed to network)
   - Servarr services are backend (typically internal only)
   - Different firewall rules possible
   - Different update schedules if needed

4. **Maintenance**
   - Update Jellyfin without affecting downloads
   - Backup strategies differ (Jellyfin: library metadata; Servarr: configs)
   - Different monitoring and logging needs

5. **Networking**
   - Can be on different Docker networks if needed
   - Can be on different machines in future
   - Jellyseerr bridges both (request → download → library)

## Linking Jellyseerr and Jellyfin (Same Network Behavior)

Even though Jellyseerr and Jellyfin run in **different compose files** and **different directories**, they behave as if they're in the same container because:

### 1. **Docker Network Connection**
```yaml
# Both services connect to the same Docker network
networks:
  - servarrnetwork  # Jellyseerr uses this

# Jellyfin also connects to this network from its compose file
networks:
  - jellyfin_network  # Bridged/linked to servarrnetwork
```

### 2. **Service Discovery via Hostname**
```bash
# From Jellyseerr's perspective:
# Instead of http://192.168.1.100:8096 (IP-based)
# Use: http://jellyfin:8096 (hostname-based)

# Docker DNS resolves "jellyfin" hostname to the Jellyfin container
# This works because both are on the same Docker network
```

### 3. **Connection Flow**

```
User Browser
    ↓
Jellyfin (http://host:8096)
    ↓
    └─→ Serves media library
    
User Request via Jellyseerr
    ↓
Jellyseerr (http://host:5055)
    ↓
    ├─→ Connects to Jellyfin (http://jellyfin:8096) ← Docker network
    ├─→ Connects to Sonarr (http://sonarr:8989) ← Docker network
    └─→ Connects to Radarr (http://radarr:7878) ← Docker network
    
Download Requests
    ↓
Sonarr/Radarr
    ↓
    ├─→ Send NZB to NZBGet ← Docker network
    └─→ Send Torrent to qBittorrent ← Docker network
    
Post-Processing
    ↓
Downloaded files → /data/downloads
    ↓
Sonarr/Radarr organize → /data/shows or /data/movies
    ↓
Jellyfin scans → Serves in library
```

### 4. **Configuration in Jellyseerr**

When setting up Jellyseerr, use **container names as hostnames**:

```yaml
# Jellyseerr Configuration (Settings → Services)

Jellyfin:
  Host: jellyfin:8096  # NOT 192.168.1.100:8096
  Username: admin
  Password: [your-password]

Sonarr:
  Host: sonarr:8989    # NOT 192.168.1.100:8989
  API Key: [your-key]

Radarr:
  Host: radarr:7878    # NOT 192.168.1.100:7878
  API Key: [your-key]
```

### 5. **Why This Works (Docker Networking)**

**Same Network (servarrnetwork):**
```
Container A (Jellyseerr) ←DNS→ Docker Network Bridge ←DNS→ Container B (Jellyfin)
                         (resolves hostname to IP)
```

**Different Compose Files (but linked via network):**
```
docker-compose -f /docker/jellyfin/compose.yaml up
docker-compose -f /docker/servarr/compose.yaml up

Both expose to the same "servarrnetwork"
DNS resolution works across compose files
```

## Implementation Steps

### Step 1: Create All Directories
```bash
# Execute the one-liner commands above
mkdir -p /data/{movies,shows,music,downloads/{nzbget/{completed,intermediate},qbittorrent/{completed,incomplete,torrents/intermediate}}}
mkdir -p /docker/{jellyfin,servarr/{sonarr,radarr,prowlarr,bazarr,jellyseerr,qbittorrent,nzbget}}

# Set permissions
chown -R 1000:1000 /data /docker
find /data -type d -exec chmod 755 {} \;
find /docker -type d -exec chmod 755 {} \;
```

### Step 2: Deploy Jellyfin First
```bash
cd /docker/jellyfin
docker-compose -f compose.yaml up -d jellyfin

# Wait for startup (30-60 seconds)
docker-compose logs -f jellyfin
```

### Step 3: Deploy Servarr Services
```bash
cd /docker/servarr
docker-compose -f compose.yaml up -d

# Monitor startup
docker-compose logs -f
```

### Step 4: Configure Services
- Open Jellyfin: `http://localhost:8096`
- Open Jellyseerr: `http://localhost:5055`
- Follow setup wizard, using container hostnames (e.g., `jellyfin:8096`)
- Configure Sonarr/Radarr with Prowlarr
- Configure download clients
- Add content

## Quick Start - One Command Deployment

If you prefer to deploy everything at once using the complete compose file, use this simplified approach:

### Prerequisites
```bash
# Create directories first
mkdir -p /data/{movies,shows,music,downloads/{nzbget/{completed,intermediate},qbittorrent/{completed,incomplete,torrents/intermediate}}}
mkdir -p /docker/{jellyfin,servarr/{sonarr,radarr,prowlarr,bazarr,jellyseerr,qbittorrent,nzbget}}

# Set permissions
chown -R 1000:1000 /data /docker
```

### Deploy All Services at Once

```bash
# Navigate to the servarr directory
cd /docker/servarr

# Start all services in the background
sudo docker compose up -d

# Monitor the startup process
docker compose logs -f
```

That's it! All services will start automatically:
- Jellyfin on port 8096
- Jellyseerr on port 5055
- Sonarr on port 8989
- Radarr on port 7878
- Prowlarr on port 9696
- qBittorrent on port 8080
- NZBGet on port 6789
- Bazarr on port 6767
- Flaresolverr on port 8191

### Verify All Services Are Running

```bash
# Check status of all containers
docker compose ps

# View logs for specific service
docker compose logs -f sonarr      # Replace 'sonarr' with any service name
docker compose logs -f radarr
docker compose logs -f jellyseerr

# Check resource usage
docker stats
```

### Stop All Services

```bash
cd /docker/servarr
sudo docker compose down
```

### Restart All Services

```bash
cd /docker/servarr
sudo docker compose restart
```

### View Logs in Real-Time

```bash
# All services
docker compose logs -f

# Specific service (5 most recent logs)
docker compose logs --tail=5 sonarr
```

## Complete Compose File Reference

The `/docker/servarr/compose.yaml` file contains all services configured to:

- **Share the servarrnetwork** - All containers communicate via Docker DNS
- **Mount /data volumes** - All have access to media storage
- **Expose ports** - Each service accessible on defined ports
- **Auto-restart** - Services restart if they crash
- **Memory limits** - Set to ~4GB per service (adjust as needed)
- **User permissions** - PGID:PUID set to 1000:1000 (adjust to match your system)

### Example Service Reference in Compose

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    ports:
      - "8989:8989"
    networks:
      - servarrnetwork      # Connects to shared network
    volumes:
      - /PATH/TO/servarr/sonarr:/config      # Config storage
      - /PATH/TO/MEDIA:/data                  # Media access
    environment:
      TZ: America/New_York
      PGID: "1000"
      PUID: "1000"
    restart: unless-stopped
```

When using `docker compose up -d`, Docker automatically:
1. Creates the servarrnetwork if it doesn't exist
2. Starts containers in dependency order
3. Enables DNS resolution between containers
4. Maps the specified ports to your host
5. Mounts the volumes to the container filesystem

## Directory Purposes Summary

| Directory | Purpose | Owner | Permissions |
|-----------|---------|-------|-------------|
| `/data/movies` | Movie library | docker user | 755 |
| `/data/shows` | TV show library | docker user | 755 |
| `/data/music` | Music library | docker user | 755 |
| `/data/downloads/nzbget` | NZBGet staging | docker user | 755 |
| `/data/downloads/qbittorrent` | qBittorrent staging | docker user | 755 |
| `/docker/jellyfin` | Jellyfin configs | docker user | 755 |
| `/docker/servarr` | Servarr configs | docker user | 755 |

## Troubleshooting

### Issue: Services can't reach each other
**Solution:** Verify both compose files use the same network name
```yaml
# Check compose files
networks:
  servarrnetwork:  # Must match across files
```

### Issue: Jellyseerr can't connect to Jellyfin
**Solution:** Use container name, not IP
```bash
# Wrong: http://192.168.1.100:8096
# Correct: http://jellyfin:8096
```

### Issue: Downloads not organizing correctly
**Solution:** Verify /data/downloads paths match compose volumes
```yaml
# Compose should have:
volumes:
  - /data/downloads/qbittorrent:/downloads  # Or your configured path
```

### Issue: Jellyfin can't see new downloads
**Solution:** Restart Jellyfin library scan
```bash
# In Jellyfin web UI:
# Settings → Library → Scan All Libraries
```
