# CasaOS Installation and Configuration Guide

CasaOS is a simple, elegant, and open-source home server operating system built on Linux and Docker. It provides a user-friendly web interface to manage applications, files, and services on your homelab without deep technical knowledge. CasaOS can serve as an alternative to raw Docker when you prefer a graphical interface.

## What is CasaOS?

**CasaOS Provides:**
- **Web UI Dashboard:** Graphical interface for managing applications and system resources
- **App Store:** Pre-configured Docker applications ready to install with one click
- **File Management:** Browse and manage files on your server
- **System Monitoring:** Watch CPU, memory, network, and disk usage
- **Easy Backup:** Simple backup and restore functionality
- **Docker Integration:** Full Docker and Docker Compose support under the hood

**CasaOS vs. Raw Docker:**
- CasaOS: Easier for beginners, graphical interface, one-click app installation
- Raw Docker: More control, faster, better for advanced users, textual configuration

For your homelab media stack, you can use CasaOS with its pre-configured media server apps or manually add your own Docker Compose stack.

## System Requirements

**Minimum Requirements:**
- Raspberry Pi 4 (4GB RAM) or better x86 system
- 2 CPU cores
- 4GB RAM
- 10GB disk space for OS and apps
- Ubuntu 20.04 LTS or later / Debian 11+

**Recommended for Media Stack:**
- x86-64 processor (Intel/AMD)
- 4+ CPU cores
- 8GB+ RAM
- 50GB+ disk space for applications and metadata
- Ubuntu 22.04 LTS
- Gigabit network connection

**Supported Platforms:**
- Raspberry Pi OS
- Ubuntu 20.04, 22.04
- Debian 11, 12
- CentOS 8+
- Other Linux distributions with Docker support

## Prerequisites Before Installation

### 1. Fresh Linux System (Recommended)

Start with a clean Ubuntu 22.04 LTS installation:

```bash
# Update system packages
sudo apt-get update
sudo apt-get upgrade -y

# Install basic tools
sudo apt-get install -y curl wget git nano
```

### 2. Snapshot/Backup Your System (Optional but Recommended)

If using a VM, take a snapshot before installation for easy rollback.

### 3. Network Configuration

Know your static IP address or configure DHCP reservation:

```bash
# View current IP
ip addr show

# Or use hostname
hostname -I
```

## Installation on Linux

### Step 1: Download and Run Installer

CasaOS provides an automated installation script:

```bash
# Download and run the installer
curl -fsSL https://get.casaos.io | sudo bash
```

This script will:
- Install Docker and Docker Compose automatically
- Install CasaOS system files
- Configure the web interface
- Start CasaOS service

### Step 2: Wait for Installation to Complete

The installation takes 2-5 minutes depending on your connection:

```bash
# Monitor installation progress
sudo journalctl -u casaos -f

# Or check service status
sudo systemctl status casaos
```

### Step 3: Verify Installation

Check that CasaOS service is running:

```bash
# Check CasaOS status
sudo systemctl status casaos

# Should show: active (running)

# Check Docker is installed
docker --version
docker compose version
```

### Step 4: Access CasaOS Web Interface

Open a web browser and navigate to CasaOS:

```
http://your-server-ip:81
```

**Finding Your Server IP:**

```bash
# Display IP address
hostname -I

# Or use this to find your device on network
ip route | grep default | awk '{print $3}'
```

**Expected:** CasaOS login page should appear

### Step 5: Initial Setup

1. **Create Admin Account:**
   - Username: (your choice, e.g., homelab)
   - Password: (strong password, store securely)
   - Click "Continue"

2. **System Name:**
   - Enter server name (e.g., "HomeServer", "MediaCenter")
   - Click "Continue"

3. **Welcome Dashboard:**
   - You're in! CasaOS dashboard appears

## Post-Installation Configuration

### 1. Configure Storage Locations

Set up directories where CasaOS will store data:

**Option A: Using CasaOS UI**

1. Click **System Settings** (bottom left gear icon)
2. Go to **Storage**
3. Add new storage paths if needed:
   - App data: `/var/lib/casaos`
   - User data: `/root/CasaOS`

**Option B: Create Directories Manually**

```bash
# Create CasaOS standard directories
sudo mkdir -p /root/CasaOS
sudo mkdir -p /var/lib/casaos

# Create media stack directories
sudo mkdir -p /docker/servarr
sudo mkdir -p /data/media/{shows,movies}
sudo mkdir -p /data/downloads

# Set permissions
sudo chown -R 1000:1000 /docker /data
sudo chmod -R 755 /docker /data
```

### 2. Configure Network Settings

Set up static IP or network settings:

1. Click **System Settings**
2. Go to **Network**
3. Configure:
   - **Network Device:** Select your primary interface (usually eth0)
   - **IP Configuration:** Set to Static or DHCP
   - If Static: enter IP, gateway, DNS servers

Example Static Configuration:
- IP Address: 192.168.1.100
- Netmask: 255.255.255.0
- Gateway: 192.168.1.1
- DNS: 8.8.8.8, 8.8.4.4

### 3. Configure System Settings

1. Click **System Settings**
2. **General:**
   - System Name: Your server name
   - Timezone: Select your timezone
   - Theme: Light or Dark mode
   - Language: Your preferred language

3. **Resources:**
   - View CPU, memory, disk usage
   - Monitor temperature if supported

4. **Applications:**
   - Auto-update settings
   - storage locations for app data

### 4. Install Docker (if not auto-installed)

Sometimes Docker isn't automatically installed. Install manually if needed:

```bash
# Check if Docker is installed
docker --version

# If not installed, follow Docker installation from docker/README.MD
```

## Installing Media Stack Applications

### Option 1: Using CasaOS App Store (Easiest)

Many media applications are available in the CasaOS App Store:

1. Click **App Store** (top of dashboard)
2. Search for applications:
   - Jellyfin (media server)
   - Sonarr (TV show manager)
   - Radarr (movie manager)
   - qBittorrent (torrent client)
3. Click app tile and select **Install**
4. Configure settings in the installation dialog
5. Click **Create** to install

**Advantages:**
- One-click installation
- Pre-configured defaults
- Easy to manage from UI

**Disadvantages:**
- Less customization than manual Docker Compose
- Version may not be the latest

### Option 2: Using Custom Docker Compose (Recommended for Control)

For advanced configuration, manually create Docker Compose stack:

**Step 1: Create Compose Directory**

```bash
sudo mkdir -p /root/CasaOS/AppData/docker-compose
cd /root/CasaOS/AppData/docker-compose
```

**Step 2: Create compose.yaml**

```bash
sudo nano compose.yaml
```

Add your media stack services (see media/README.MD for service configurations).

**Step 3: Deploy Stack via CasaOS**

1. Click **Settings** (gear icon)
2. Go to **Docker** or **Compose**
3. Upload or paste your compose.yaml file
4. Click **Deploy** or **Create**

**Step 4: Monitor Deployment**

1. Click **Docker** in main navigation
2. See all running containers and their status
3. View logs by clicking on container

### Option 3: Using Terminal with CasaOS

CasaOS still allows terminal access for advanced users:

```bash
# Access terminal via SSH
ssh username@server-ip

# Or use CasaOS terminal if available (check settings)

# Navigate to compose directory
cd /root/CasaOS/AppData/docker-compose

# Start services
docker compose up -d

# View status
docker compose ps
```

## Managing Applications in CasaOS

### View Running Applications

**Dashboard:**
- Main view shows all installed applications
- Click app tile to access its web interface
- See resource usage per application

### Container Status

1. Click **Docker** in navigation
2. View all running containers:
   - Green = Running
   - Yellow = Starting/Stopping
   - Red = Stopped/Error

3. Click container for:
   - Logs (troubleshooting)
   - Statistics (CPU, memory usage)
   - Configuration

### Stop/Start Applications

```bash
# Via CasaOS UI: Click container, select Stop/Restart

# Via terminal:
docker compose stop service_name
docker compose start service_name
docker compose restart service_name
```

### View Application Logs

1. Click **Docker** in navigation
2. Find the application container
3. Click on it to view logs
4. Search and filter logs as needed

## Port Management

CasaOS applications use specific ports. Make sure they don't conflict:

**Common Media Stack Ports:**
- 6789: NZBGet
- 6767: Bazarr
- 7878: Radarr
- 8080: qBittorrent
- 8096: Jellyfin
- 8989: Sonarr
- 9696: Prowlarr
- 5055: Jellyseerr

**Check if Port is in Use:**

```bash
sudo netstat -tlnp | grep :8096
```

**Configure Ports in CasaOS:**
1. When installing app, set port mappings
2. Click app settings to change ports if needed
3. Ensure ports don't duplicate

## File Management in CasaOS

### Access Files via Web UI

1. Click **Files** in main navigation
2. Browse `/root/CasaOS` directory structure
3. Upload, download, or manage files
4. Create directories for media organization

### Upload Media Files

```bash
# Via SFTP/SSH:
sftp username@server-ip

# Or use GUI file manager to connect via SMB/SFTP

# Via terminal:
scp /local/path/to/file username@server-ip:/root/CasaOS/path/
```

## Backup and Restore

### Backup CasaOS

1. Click **System Settings** (gear)
2. Go to **Backup**
3. Click **Backup Now**
4. Download backup file (saves application data and configurations)

### Restore from Backup

1. Click **System Settings**
2. Go to **Backup**
3. Upload previous backup file
4. Click **Restore**
5. CasaOS will restore all applications and data

## Accessing CasaOS Remotely

### Option 1: Port Forward (Least Secure)

1. In your router, forward port 81 to your server
2. Access via `http://your-public-ip:81`
3. Login with your CasaOS credentials

**Security Risk:** Exposes CasaOS web interface to internet attacks

### Option 2: VPN (Recommended)

1. Set up WireGuard or OpenVPN on your server
2. Connect to VPN from remote location
3. Access CasaOS on local network via `http://local-ip:81`

**Security:** VPN encrypts all traffic, server not exposed

### Option 3: Reverse Proxy

1. Use nginx or Caddy as reverse proxy
2. Map external domain to CasaOS port 81
3. Use HTTPS with Let's Encrypt certificate
4. Access via `https://your-domain.com`

## Updating CasaOS and Applications

### Auto-Update CasaOS

1. Go to **System Settings**
2. Scroll to **About** section
3. Enable **Auto-update** if available
4. Or click **Check for Updates**

### Manual Update

```bash
# Update system packages
sudo apt-get update && sudo apt-get upgrade -y

# Update Docker containers
docker compose pull
docker compose up -d

# CasaOS will auto-update service
```

### Update Individual Applications

1. Go to **Docker**
2. Select container
3. Pull latest image: `docker pull image_name`
4. Recreate container with new image

## System Monitoring

### View System Resources

1. Click **System Settings**
2. Go to **Resources** tab
3. See real-time:
   - CPU usage and temperature
   - Memory and swap usage
   - Disk usage by partition
   - Network traffic

### Monitor Containers

1. Click **Docker**
2. See all containers with status
3. Click individual containers for:
   - CPU and memory usage
   - Network traffic
   - Uptime and restarts

## Network and Connectivity

### Check Connectivity

```bash
# Check internet connection
ping 8.8.8.8

# Check DNS resolution
nslookup google.com

# Check container connectivity
docker exec container_name ping other_container
```

### View Network Configuration

```bash
# Show IP address
hostname -I

# View network interfaces
ip link show

# View routing table
ip route show
```

## Troubleshooting

### CasaOS Won't Start

**Symptom:** Cannot access web interface at port 81

```bash
# Check service status
sudo systemctl status casaos

# Check if port is in use
sudo netstat -tlnp | grep :81

# Restart service
sudo systemctl restart casaos

# Check logs
sudo journalctl -u casaos -n 50
```

### Docker Not Running

**Symptom:** Containers won't start or show as stopped

```bash
# Check Docker status
sudo systemctl status docker

# Start Docker
sudo systemctl start docker

# Check Docker info
docker info
```

### Application Won't Start

**Symptom:** Container shows as stopped or in error state

```bash
# Check container logs
docker logs container_name

# Check exit code
docker inspect container_name | grep -A 5 "ExitCode"

# Try restarting
docker restart container_name

# Check resource usage
docker stats container_name
```

### Port Conflicts

**Symptom:** Application won't start - "port already in use"

```bash
# Find what's using the port
sudo netstat -tlnp | grep :port_number

# Kill the process if needed
sudo kill -9 process_id

# Or change port in compose.yaml and restart
```

### Out of Disk Space

**Symptom:** Applications fail to start or save data

```bash
# Check disk usage
df -h

# Find largest files
du -sh /* | sort -rh

# Clean up Docker
docker system prune -a

# View Docker storage
docker system df
```

### Slow Performance

**Symptom:** CasaOS or containers running slowly

```bash
# Check system resources
top

# Check disk I/O
iostat -x 1 5

# Check network
iftop

# Check Docker resource limits
docker stats

# Restart services
docker compose restart
```

## Advanced Configuration

### Enable SSH Access

1. Go to **System Settings**
2. Look for **SSH** or **Remote Access**
3. Enable SSH service
4. Note the port (usually 22)

Connect via terminal:
```bash
ssh user@server-ip
```

### Custom Domain Setup

1. Register domain or use dynamic DNS
2. Point domain to your server's IP
3. Set `JELLYFIN_PublishedServerUrl` in Jellyfin settings
4. Test external access

### Hardware Acceleration (GPU)

Enable GPU support for transcoding:

```bash
# Check GPU availability
ls -la /dev/dri/

# Add to compose.yaml for Jellyfin or other services:
devices:
  - /dev/dri:/dev/dri
```

## Uninstalling CasaOS

If you need to remove CasaOS:

```bash
# Stop CasaOS service
sudo systemctl stop casaos

# Remove CasaOS
sudo rm -rf /var/lib/casaos
sudo rm -rf /root/CasaOS

# Disable service
sudo systemctl disable casaos

# Note: Docker and containers remain installed
# To remove Docker entirely, follow Docker uninstall guide
```

## Next Steps After Installation

1. **Configure Storage:** Set up `/docker/servarr` and `/data/media` directories
2. **Install Applications:** Use App Store or custom Docker Compose
3. **Configure Services:** Follow individual service guides (Sonarr, Radarr, Jellyfin, etc.)
4. **Set Up Users:** Create accounts for other family members
5. **Configure Backup:** Enable regular CasaOS backups
6. **Monitor Performance:** Watch resource usage as services load

## Comparison: CasaOS vs. Manual Docker

| Feature | CasaOS | Manual Docker |
|---------|--------|---------------|
| Learning Curve | Easy - GUI | Moderate - CLI |
| Installation Speed | Very Fast (app store) | Faster with templates |
| Customization | Limited | Full control |
| Resource Usage | Slightly higher (UI overhead) | Lower |
| Management | Web UI dashboard | CLI + text config |
| Scalability | Good for 10-20 apps | Better for 20+ apps |
| Best For | Beginners, Home users | Advanced users |

## Resources and Support

**CasaOS Official:**
- Website: https://casaos.io
- GitHub: https://github.com/IceWhaleTech/CasaOS
- Community: https://forum.casaos.io
- Docs: https://docs.casaos.io

**Media Stack Applications:**
- See individual README files in media/ folder
- Each service has detailed configuration guides
