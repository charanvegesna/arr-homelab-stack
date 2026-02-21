# Docker Installation and Configuration Guide

Docker is a containerization platform that packages applications and their dependencies into isolated containers. For your homelab media stack, Docker enables easy deployment and management of services like Sonarr, Radarr, Jellyfin, and others without complex system-wide configurations.

## Why Docker for Homelab?

- **Isolation:** Each service runs independently, preventing software conflicts
- **Portability:** Same setup works across different machines and operating systems
- **Easy Management:** Start, stop, and update services with simple commands
- **Resource Efficiency:** Lightweight containers vs heavy virtual machines
- **Clean Uninstallation:** Remove services completely without leftover system files
- **Version Control:** Easy to version and roll back service updates

## Prerequisites

- Ubuntu 22.04 LTS or similar Linux x86_64 system (recommended for homelab)
- Internet connection for downloading Docker packages
- sudo or root access to install system packages
- Terminal/SSH access to your server

## Installation on Linux (Ubuntu/Debian)

### Step 1: Remove Old Docker Versions (if applicable)

If you have an older Docker installation, remove it first:

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc -y 2>/dev/null || true
```

### Step 2: Update System Packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### Step 3: Install Required Dependencies

These packages are needed to set up Docker's secure repository:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### Step 4: Add Docker's Official GPG Key

This ensures packages are verified and authentic:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### Step 5: Add Docker Repository

**For Ubuntu 22.04:**

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**For Debian 11+:**

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 6: Install Docker Engine and Docker Compose

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Step 7: Verify Docker Installation

```bash
sudo docker --version
sudo docker run hello-world
```

**Expected output:**
- Docker version 20.10 or higher
- Hello from Docker! message confirming installation

## Post-Installation Configuration (Important)

### 1. Configure User Permissions

By default, Docker requires sudo. To run Docker commands without sudo:

```bash
# Create docker group (usually already exists)
sudo groupadd docker 2>/dev/null || true

# Add current user to docker group
sudo usermod -aG docker $USER

# Apply group membership - choose one method:
# Option 1: Re-login to your account (most reliable)
# Option 2: Use newgrp command
newgrp docker

# Verify it works (should work without sudo)
docker ps
```

### 2. Configure Docker Daemon (Recommended)

Edit Docker's configuration file to optimize for homelab use:

```bash
sudo nano /etc/docker/daemon.json
```

Paste the following configuration:

```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "live-restore": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 262144,
      "Soft": 262144
    }
  }
}
```

**Configuration Explanation:**
- `storage-driver`: overlay2 - modern, efficient storage backend for containers
- `log-driver`: json-file - structured JSON logging
- `log-opts`: Prevent Docker logs from filling disk (max 10MB per log, 3 files)
- `userland-proxy`: false - better network performance for homelab
- `live-restore`: true - containers keep running if Docker daemon crashes
- `default-ulimits`: Increase file descriptor limits for applications needing many open files

Apply the configuration:

```bash
sudo systemctl restart docker
```

Verify changes were applied:

```bash
docker info | grep -A 10 "Storage Driver"
```

### 3. Enable Docker Service Auto-Start

Services restart automatically if server reboots:

```bash
sudo systemctl enable docker
sudo systemctl enable containerd
```

Verify auto-start is enabled:

```bash
sudo systemctl is-enabled docker
```

Should output "enabled".

### 4: Create Docker Network for Services

All services in your media stack communicate through a shared network:

```bash
docker network create servarrnetwork
```

Verify the network was created:

```bash
docker network ls
```

Should show "servarrnetwork" in the list.

## Directory Structure Preparation

### Create Directories for Media Stack

Set up the directory structure your entire media stack will use:

```bash
# Create Docker service config directory
sudo mkdir -p /docker/servarr

# Create media and download directories
sudo mkdir -p /data/media/shows
sudo mkdir -p /data/media/movies
sudo mkdir -p /data/downloads/nzbget/{completed,intermediate}
sudo mkdir -p /data/downloads/qbittorrent/{completed,incomplete}
sudo mkdir -p /data/downloads/combined

# Set ownership (adjust 1000:1000 to match your user - see note below)
sudo chown -R 1000:1000 /docker
sudo chown -R 1000:1000 /data

# Set permissions
sudo chmod -R 755 /docker
sudo chmod -R 755 /data
```

**Finding Your User ID and Group ID:**

```bash
# Find your user and group IDs
id $USER

# Output should show: uid=1000(username) gid=1000(username)
# Use these numbers in your service configurations
```

## Verify Complete Setup

Run these verification commands to ensure everything is properly configured:

### 1. Check Versions

```bash
docker --version          # Should show v20.10+
docker compose version    # Should show v2.0+
```

### 2. Check Docker Info

```bash
docker info
```

**Look for:**
- Storage Driver: overlay2
- Live Restore Enabled: true
- Cgroup Driver: systemd (or cgroupfs)

### 3. Verify Network

```bash
docker network ls | grep servarrnetwork
```

Should show the network in the list.

### 4. Verify User Permissions

```bash
docker ps
```

Should work without sudo and show no containers.

### 5. Test Docker Run

```bash
docker run --rm alpine echo "Docker is working!"
```

Should print "Docker is working!" and exit cleanly.

## Docker Compose Setup

### Create Base Compose File Structure

```bash
# Navigate to Docker directory
cd /docker/servarr

# Create base compose.yaml
nano compose.yaml
```

Paste this template:

```yaml
version: "3.8"

services:
  # Services will be added here
  # Examples: sonarr, radarr, jellyfin, prowlarr, qbittorrent, nzbget, etc.

networks:
  servarrnetwork:
    external: true
```

Save with Ctrl+X, Y, Enter.

### Verify Compose Works

```bash
cd /docker/servarr
docker compose config
```

Should output the YAML config without errors.

## Basic Docker Commands Reference

### Container Management

```bash
# List running containers
docker ps

# List all containers (running and stopped)
docker ps -a

# View logs from a container (live stream with -f)
docker logs -f container_name

# Start/stop a container
docker start container_name
docker stop container_name

# Restart a container
docker restart container_name

# Remove a container
docker rm container_name

# View container stats (CPU, memory, network)
docker stats container_name
```

### Docker Compose Commands

```bash
# Start all services defined in compose.yaml
docker compose up -d

# Stop all services
docker compose down

# View service status
docker compose ps

# View logs from a specific service
docker compose logs -f service_name

# Restart a service
docker compose restart service_name

# Re-create services (pull latest images)
docker compose up -d
```

### System Management

```bash
# View Docker disk usage
docker system df

# Remove unused resources (cleanup)
docker system prune -a

# Remove dangling images
docker image prune -a

# Check Docker storage location
docker info | grep "Docker Root Dir"
```

## Hardware Acceleration (Optional but Recommended)

### Enable VAAPI (Intel/AMD GPU Acceleration)

If transcoding will be used (Jellyfin for video conversion):

```bash
# Check if VAAPI is available on your system
ls -la /dev/dri/

# Should show devices like renderD128
```

If devices exist, you can enable GPU acceleration in service compose files by adding:

```yaml
services:
  jellyfin:
    devices:
      - /dev/dri:/dev/dri  # GPU device passthrough
```

## Troubleshooting

### Docker Daemon Not Running

**Symptom:** `Cannot connect to Docker daemon`

```bash
# Check Docker status
sudo systemctl status docker

# Start Docker
sudo systemctl start docker

# Check logs if it fails to start
sudo journalctl -u docker -n 50
```

### Permission Denied

**Symptom:** `permission denied while trying to connect to Docker daemon`

```bash
# Re-add user to docker group
sudo usermod -aG docker $USER

# Apply group changes
newgrp docker

# Verify
docker ps
```

### Port Already in Use

**Symptom:** `bind: address already in use`

```bash
# Find what's using the port (example: 8096)
sudo netstat -tlnp | grep 8096

# Kill the process
sudo kill -9 process_id
```

### Disk Space Full

**Symptom:** `no space left on device`

```bash
# Check disk usage
df -h

# Clean up unused Docker resources
docker system prune -a

# View detailed Docker storage
docker system df

# Check largest images
docker images --format "table {{.Repository}}\t{{.Size}}" | sort -k2 -hr | head -10
```

### Container Crashes Immediately

**Symptom:** Container starts then stops without running

```bash
# Check container logs
docker logs container_name

# Check exit code
docker inspect container_name | grep -A 5 "ExitCode"

# Common causes:
# - Port mapping conflicts
# - Missing environment variables
# - Volume mount permission issues
# - Insufficient system resources
```

### Services Can't Communicate (Network Issues)

**Symptom:** Services can't reach each other by container name

```bash
# Verify network exists
docker network ls | grep servarrnetwork

# Check container is on network
docker inspect container_name | grep -A 20 "Networks"

# Test connectivity
docker exec container_name ping other_container_name
```

### Docker Logs Growing Too Large

Even with log rotation configured, logs may still grow:

```bash
# Check Docker log directory size
sudo du -sh /var/lib/docker/containers

# Clear logs for running container
docker container prune -f

# Manually truncate large logs
truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

## Next Steps

After Docker is installed and configured:

1. **Verify Setup:** Run all verification commands above
2. **Create Compose File:** Use the template in Docker Compose Setup section
3. **Read Service Guides:** Follow documentation for each service (Sonarr, Radarr, Jellyfin, etc.)
4. **Add Services:** Add service blocks to your compose.yaml file
5. **Start Stack:** Run `docker compose up -d` to start all services

All service documentation assumes Docker and Docker Compose are installed and configured as described in this guide.

## Installation on Other Platforms

### macOS with Docker Desktop

1. Download Docker Desktop from `https://www.docker.com/products/docker-desktop`
2. Open installer and drag Docker to Applications folder
3. Launch Docker from Applications
4. Verify: `docker --version`

**Note:** Docker Desktop on macOS runs a hidden Linux VM. Performance suitable for development but homelab typically runs on Linux servers.

### Windows with Docker Desktop

Similar to macOS - download Docker Desktop for Windows and install. Requires Windows 10 Pro or higher.

**Note:** Most homelab setups use Linux servers, not Windows, for Docker containers.

## Security Recommendations

### 1. Firewall Configuration

Only expose necessary ports:

```bash
# Enable UFW firewall
sudo ufw enable

# Allow SSH (if remote)
sudo ufw allow 22/tcp

# Allow specific services (examples)
sudo ufw allow 8096/tcp    # Jellyfin
sudo ufw allow 9696/tcp    # Prowlarr
sudo ufw allow 8989/tcp    # Sonarr

# Check rules
sudo ufw status
```

### 2. Regular Updates

Keep Docker and images current:

```bash
# Update Docker packages
sudo apt-get update && sudo apt-get upgrade -y

# Pull latest images
docker pull image_name

# Recreate containers with new images
docker compose up -d
```

### 3. Resource Limits

Prevent containers from consuming all resources:

Add to each service in compose.yaml:

```yaml
services:
  service_name:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
```

This limits the service to 2 CPU cores and 2GB RAM maximum.
