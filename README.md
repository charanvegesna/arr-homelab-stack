# ARR Homelab Stack

Overview
-
This repository documents a small home server used to run media and auxiliary services (Jellyfin, other containerized media tooling, and supporting infrastructure). It contains deployment manifests, docs, and utilities to run the stack on an older server or laptop.

Repository Layout
-

**Infrastructure:**
- `infrastructure/docker/` — [Docker installation and setup guide](infrastructure/docker/README.MD) for containerization
- `infrastructure/CasaOS/` — [CasaOS installation guide](infrastructure/CasaOS/README.MD) for graphical container management
- `infrastructure/Tailscale/` — [Tailscale setup guide](infrastructure/Tailscale/README.md) for secure remote access

**Media Services:**
- `media/compose.yaml` — Main Docker Compose aggregator for all media services
- `media/Jellyfin/` — [Jellyfin media server guide](media/Jellyfin/README.MD) for streaming your library
- `media/Sonarr/` — [Sonarr guide](media/Sonarr/README.MD) for auto-managing TV shows
- `media/Radarr/` — [Radarr guide](media/Radarr/README.MD) for auto-managing movies
- `media/Prowlarr/` — [Prowlarr guide](media/Prowlarr/README.MD) for managing indexers
- `media/qbittorrent/` — [qBittorrent guide](media/qbittorrent/README.MD) for torrent downloads
- `media/nzbget/` — [NZBGet guide](media/nzbget/README.MD) for usenet downloads
- `media/Jellyseerr/` — [Jellyseerr guide](media/Jellyseerr/README.MD) for content requests
- `media/Bazarr/` — [Bazarr guide](media/Bazarr/README.MD) for subtitle management
- `media/Flaresolverr/` — [Flaresolverr guide](media/Flaresolverr/README.MD) for Cloudflare bypass

**Installation:**
- `INSTALL_UBUNTU.md` — Step-by-step guide to install Ubuntu on older hardware and configure SSH

What This Server Is
-
This machine is a compact homelab/media server that automates downloading, organizing, and streaming your personal media library. **Key responsibilities:**

**Media Delivery:**
- Host Jellyfin media server for secure streaming to devices (LAN and remote via Tailscale)
- Auto-download TV shows (Sonarr) and movies (Radarr) with quality management
- Download from torrents (qBittorrent) or usenet (NZBGet) via automated indexing (Prowlarr)

**Content Management:**
- Auto-fetch and manage subtitles (Bazarr)
- Handle user requests for new content (Jellyseerr)
- Bypass protection services blocking indexer access (Flaresolverr)

**Infrastructure:**
- Run all services in Docker containers for isolation and easy updates
- Secure remote access to manage or watch from anywhere (Tailscale VPN)
- Manage containers via Docker CLI or CasaOS web UI

Architecture Overview
-

```
Internet / Remote Devices
    ↓ (secured via Tailscale VPN)
Docker Host (Linux server with Docker)
    ├─ Download Clients
    │  ├─ qBittorrent (torrents)
    │  └─ NZBGet (usenet)
    │
    ├─ Management & Organization
    │  ├─ Prowlarr (indexer hub)
    │  ├─ Flaresolverr (bypass protection)
    │  ├─ Sonarr (auto TV shows)
    │  ├─ Radarr (auto movies)
    │  └─ Bazarr (auto subtitles)
    │
    ├─ User Interfaces
    │  ├─ Jellyfin (watch media)
    │  └─ Jellyseerr (request new content)
    │
    └─ Storage
       ├─ /data/downloads (temp)
       └─ /data/media (library)
```

**Technology Stack:**
- **Containerization:** Docker & Docker Compose for all services
- **Management:** Docker CLI (or CasaOS GUI if preferred)
- **Networking:** Custom Docker network (servarrnetwork) for service communication
- **Remote Access:** Tailscale for secure VPN without port forwarding
- **Storage:** Host volumes mounted into containers

Quick Start Guide
-

**1. Prerequisites:**
- Linux server (Ubuntu 20.04+ recommended) — see [INSTALL_UBUNTU.md](INSTALL_UBUNTU.md)
- [Docker installed](infrastructure/docker/README.MD)
- SSH access to your server (for remote management)

**2. Deploying the Stack (quick, safe steps):**

Follow these steps on the server where you will run the containers. The examples assume you will place the repository in `homelab-stack` and run Compose from the `media` folder.

```bash
# 1) (Optional) SSH to your server
# ssh user@your-server-ip

# 2) Clone repo and enter the media directory
git clone https://github.com/<owner>/<repo>.git homelab-stack
cd homelab-stack/media

# 3) Edit compose.yaml before starting
# - Set host volume paths (media, downloads, configs)
# - Check ports, UID/GID and any env vars in each service's section
# Use your editor: e.g. nano, vim, or VS Code
# vim compose.yaml

# 4) Start services (Docker Compose v2 syntax)
docker compose up -d

# If your system uses the legacy plugin, use:
# docker-compose up -d

# 5) Verify running services
docker compose ps

# 6) Helpful service commands:
# Start only specific services: docker compose up -d jellyfin prowlarr
# Follow logs for a service: docker compose logs -f jellyfin
# Stop services: docker compose down
```

Notes:
- Edit `compose.yaml` to match your host directory layout (see "Configure Storage Paths").
- If you prefer a GUI, you can manage containers with CasaOS instead of the CLI.
- Start one service at a time (for example, `jellyfin`) if you want to validate configuration before bringing up the whole stack.

**3. Start with Jellyfin (Central Media Hub):**
First, set up your media server: [Jellyfin Setup Guide](media/Jellyfin/README.MD)
- This is the core service where all your media will be streamed from
- Get it running and test playback before configuring downloaders

**4. Configure Downloaders & Managers:**
Set up the automation pipeline in this order for each download type:

**For Movies (via Torrents):**
- [Prowlarr](media/Prowlarr/README.MD) (indexer hub) → [qBittorrent](media/qbittorrent/README.MD) (download) → [Radarr](media/Radarr/README.MD) (auto-manage)

**For TV Shows (via Torrents or Usenet):**
- [Prowlarr](media/Prowlarr/README.MD) (indexer hub) → [NZBGet](media/nzbget/README.MD) or [qBittorrent](media/qbittorrent/README.MD) (download) → [Sonarr](media/Sonarr/README.MD) (auto-manage)

**For Subtitles:**
- [Bazarr](media/Bazarr/README.MD) (auto-download subtitles for Sonarr/Radarr content)

**Protection & Requests:**
- [Flaresolverr](media/Flaresolverr/README.MD) (bypass Cloudflare/protection on indexers)
- [Jellyseerr](media/Jellyseerr/README.MD) (let users request new content)

**4. Setup Remote Access (Tailscale):**
Follow the [Tailscale setup guide](infrastructure/Tailscale/README.md) to access your homelab securely from anywhere without port forwarding.

**5. Configure Storage Paths:**
Edit compose files to match your host directory structure:
```bash
# Example directory structure
/docker/servarr/*     # Service configs
/data/media/shows     # TV shows
/data/media/movies    # Movies
/data/downloads       # Temp download location
```

See individual service guides for detailed path requirements.

Service Reference
-

| Service | Port | Purpose | Setup Guide |
|---------|------|---------|-------------|
| **Jellyfin** | 8096 | Media server (watch your library) | [Guide](media/Jellyfin/README.MD) |
| **Jellyseerr** | 5055 | Request management (users request content) | [Guide](media/Jellyseerr/README.MD) |
| **Sonarr** | 8989 | Auto TV shows (download & organize) | [Guide](media/Sonarr/README.MD) |
| **Radarr** | 7878 | Auto movies (download & organize) | [Guide](media/Radarr/README.MD) |
| **Prowlarr** | 9696 | Indexer hub (manage torrent/usenet sites) | [Guide](media/Prowlarr/README.MD) |
| **qBittorrent** | 8080 | Torrent client (download torrents) | [Guide](media/qbittorrent/README.MD) |
| **NZBGet** | 6789 | Usenet client (download usenet) | [Guide](media/nzbget/README.MD) |
| **Bazarr** | 6767 | Auto subtitles (download subtitles) | [Guide](media/Bazarr/README.MD) |
| **Flaresolverr** | 8191 | Cloudflare bypass (unblock indexers) | [Guide](media/Flaresolverr/README.MD) |

**Accessing Services:**
- **Local Network (LAN):** `http://server-ip:PORT` (e.g., `http://192.168.1.100:8096`)
- **Remote (via Tailscale):** `http://server-tailscale-name:PORT` (e.g., `http://homeserver:8096`)

See [Tailscale guide](infrastructure/Tailscale/README.md) for secure remote access setup.

---

OS Installation & SSH
-
If you need to install the OS on an older server or laptop, follow [INSTALL_UBUNTU.md](INSTALL_UBUNTU.md) for creating a bootable USB, the installation steps, and configuring SSH access.

Documentation Status
-

✅ **Complete & Documented:**
- All media services have comprehensive guides with setup, configuration, and troubleshooting
- Docker installation and setup guide
- CasaOS GUI alternative guide
- Tailscale secure remote access guide
- Ubuntu installation guide for older hardware

📚 **What Each Guide Includes:**
- Directory structure and file organization
- Docker Compose configuration (ready to use)
- Initial setup and configuration walkthrough
- Integration with other services in the stack
- Troubleshooting common issues
- Service-specific features and advanced options

---

Credits
-
This setup was inspired by and reuses ideas from the TechHutTV homelab project: https://github.com/TechHutTV/homelab — thanks to the authors and community for their guidance.

