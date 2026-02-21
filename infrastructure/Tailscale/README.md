# Tailscale Installation and Setup Guide

Tailscale is a modern VPN service that securely connects your devices and allows you to access your homelab services from anywhere. Unlike traditional VPNs, Tailscale uses WireGuard technology for fast, encrypted connections and doesn't require complex server setup.

## What is Tailscale?

**Tailscale Provides:**
- **Private VPN Network:** Secure mesh network between all your devices
- **Zero Configuration:** No port forwarding or server setup needed
- **Fast Connections:** Uses WireGuard for low-latency performance
- **Cross-Platform:** Works on Linux, Windows, macOS, iOS, Android
- **Free for Personal Use:** Home users can connect up to 100 devices free
- **Device Approval:** Control which devices can join your network
- **Secure DNS:** Optional ad-blocking and security features

**Why Use Tailscale for Homelab?**
- Access Jellyfin, Radarr, Sonarr from anywhere without public internet exposure
- Secure connection - no port forwarding necessary
- All traffic encrypted end-to-end
- Simple device management through web dashboard
- Works across different networks (home, mobile, remote office)

## How It Works

Tailscale creates a private network where your server and remote devices can securely communicate:

```
Remote Device (Phone/Laptop)
    ↓
Tailscale Network (Encrypted)
    ↓
Your Server at Home
    ↓
Jellyfin, Sonarr, Radarr, etc.
```

When connected to Tailscale, all devices have private IP addresses (100.x.x.x range) and can communicate securely without exposing ports to the internet.

## Prerequisites

**For Your Server:**
- Linux (Ubuntu 20.04+, Debian 11+) OR Windows/macOS
- Internet connection
- sudo or root access for installation

**For Remote Devices:**
- Any device running: Linux, Windows, macOS, iOS, or Android
- Internet connection
- Free Tailscale account (sign up at tailscale.com)

## Installation on Linux Server

### Step 1: Add Tailscale Repository

Install Tailscale's GPG key and repository:

```bash
# Add Tailscale's GPG key
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.gpg | sudo apt-key add -

# Add Tailscale repository
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.list | sudo tee /etc/apt/sources.list.d/tailscale.list > /dev/null
```

**For Debian instead of Ubuntu:**

```bash
# Add Tailscale repository for Debian
curl -fsSL https://pkgs.tailscale.com/stable/debian/bullseye.list | sudo tee /etc/apt/sources.list.d/tailscale.list > /dev/null
```

### Step 2: Update Package Lists

```bash
sudo apt-get update
```

### Step 3: Install Tailscale

```bash
sudo apt-get install tailscale -y
```

### Step 4: Start Tailscale Service

```bash
# Start the Tailscale daemon
sudo systemctl start tailscale

# Enable auto-start on boot
sudo systemctl enable tailscale

# Verify it's running
sudo systemctl status tailscale
```

### Step 5: Authenticate with Tailscale

Authenticate your server with your Tailscale account:

```bash
sudo tailscale up
```

This will output a URL like: `https://login.tailscale.com/a/1a2b3c4d5e6f`

**In your web browser:**
1. Open that URL
2. Sign in with your Google, Microsoft, or GitHub account
3. Authorize your server
4. You'll see: "Logged in as [username]"

### Step 6: Verify Connection

```bash
# Check your Tailscale IP address
tailscale ip -4

# Should output something like: 100.64.x.x

# Verify connectivity
ping 100.64.x.x
```

Your server now has a private Tailscale IP address and is ready to accept connections from other devices.

## Installation on Other Platforms

### macOS Server

```bash
# Install via Homebrew
brew install tailscale

# Start Tailscale
brew services start tailscale

# Authenticate
tailscale up

# Get your IP
tailscale ip -4
```

### Windows Server

1. Download from: `https://tailscale.com/download/windows`
2. Run installer (.msi file)
3. Follow installation wizard
4. Login with your Tailscale account
5. Allow through firewall if prompted

### Docker Container (Optional - for containerized Tailscale)

Add to your compose.yaml if you want Tailscale in a container:

```yaml
tailscale:
  image: tailscale/tailscale:latest
  container_name: tailscale
  hostname: tailscale
  environment:
    TS_AUTHKEY: "tskey-your-key-here"  # Generate from Tailscale account
    TS_EXTRA_ARGS: "--advertise-routes=10.0.0.0/8"  # Optional: for LAN access
  volumes:
    - /var/lib/tailscale:/var/lib/tailscale
    - /dev/net/tun:/dev/net/tun
  cap_add:
    - NET_ADMIN
    - SYS_MODULE
  restart: unless-stopped
```

## Setting Up Remote Access Devices

Once your server is connected, add your personal devices to the Tailscale network.

### Mobile Device (iOS/Android)

1. Download Tailscale app from App Store or Google Play
2. Open app and tap "Sign In"
3. Select your auth provider (Google, GitHub, Microsoft)
4. Review and accept the authorization
5. You're now connected to your Tailscale network

### Windows/macOS Computer

1. Download from `https://tailscale.com/download`
2. Install the application
3. Click "Sign In"
4. Authorize with your account
5. System tray icon shows connection status

### Linux Desktop/Laptop

Same as Linux server installation, then run `sudo tailscale up` and authenticate.

## Accessing Your Homelab Services

Once connected to Tailscale, you can access all your homelab services securely.

### Find Your Server's Tailscale IP

```bash
# On your server
tailscale ip -4

# Output will be something like: 100.64.123.45
```

### Access Services via Tailscale IP

**Jellyfin Media Server:**
```
http://100.64.123.45:8096
```

**Sonarr TV Show Manager:**
```
http://100.64.123.45:8989
```

**Radarr Movie Manager:**
```
http://100.64.123.45:7878
```

**Prowlarr Indexer Manager:**
```
http://100.64.123.45:9696
```

**Jellyseerr Request Manager:**
```
http://100.64.123.45:5055
```

Replace `100.64.123.45` with your actual Tailscale IP.

### Access via Device Name (Easier)

Instead of remembering IP addresses, use device names:

**On your server, set a device name:**

```bash
sudo tailscale set --hostname=homeserver
```

**Now access from remote devices:**
```
http://homeserver:8096         # Jellyfin
http://homeserver:8989         # Sonarr
http://homeserver:7878         # Radarr
```

Much easier to remember!

## Configuration and Management

### View Connected Devices

```bash
# List all devices on your Tailscale network
tailscale status

# Sample output:
# homeserver              100.64.55.123    active; relay "sfo"
# samsung-phone           100.65.42.87     active
# macbook-pro             100.72.15.33     active
```

### Configure DNS Settings (Optional)

Enable Tailscale's built-in DNS for ad-blocking:

```bash
sudo tailscale up --accept-dns=true

# Or use MagicDNS to resolve device names
sudo tailscale up --accept-dns=true --advertise-exit-node=false
```

### Allow LAN Access Through Tailscale

If you want to access other devices on your home network through Tailscale:

```bash
# Advertise your local network through Tailscale
sudo tailscale up --advertise-routes=192.168.1.0/24

# Then approve the route in Tailscale admin panel
```

### Disable Tailscale Temporarily

```bash
# Stop but keep settings
sudo tailscale down

# Resume with same settings
sudo tailscale up
```

## Tailscale Admin Panel

Manage your Tailscale network at: `https://login.tailscale.com`

### Features Available:

**Devices:**
- View all connected devices
- See last seen time
- Disable/remove devices
- Set device names

**Access Controls:**
- Define which devices can see which devices
- Set ACL rules (allow/deny connections)
- Default: all devices can see all devices

**DNS:**
- Configure Tailscale DNS
- Enable MagicDNS for device names
- Split DNS for specific domains

**Settings:**
- General account settings
- Payment info (if using paid features)
- API/webhooks configuration

## Advanced: Using Tailscale with Docker

### Run Tailscale in Docker

For containerized deployment, mount Tailscale:

```yaml
services:
  jellyfin:
    image: nyanmisaka/jellyfin:latest
    container_name: jellyfin
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: "America/New_York"
    # ... other config ...
    network_mode: "host"  # Required to use host's Tailscale
    depends_on:
      - tailscale
```

### Verify Docker Containers Can See Tailscale

```bash
# From inside a container
docker exec container_name curl http://100.64.x.x:8096

# Should work if Tailscale is accessible
```

## Security Considerations

### 1. Keep Tailscale Updated

```bash
# Check for updates
sudo apt-get update && sudo apt-get upgrade tailscale -y

# Service auto-updates on Linux
```

### 2. Device Approval

When a new device connects, it appears in your Tailscale admin panel:

1. Go to `https://login.tailscale.com/admin/machines`
2. Review pending devices
3. Click device to approve or remove

### 3. ACL (Access Control Lists)

Control which devices can communicate:

1. Go to Admin Panel → Access Controls
2. Edit policy in editor
3. Example policy:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:members"],
      "dst": ["autogroup:self:admin"]
    }
  ]
}
```

### 4. Disable Unnecessary Devices

Remove devices you no longer use from admin panel to maintain security.

### 5. Use Strong Passwords

Your Tailscale account protects access to your entire homelab. Use:
- Strong password
- Two-factor authentication (if available)
- Password manager to store credentials

## Performance on Tailscale

### Optimize Connection Quality

**Check Connection Quality:**

```bash
# From a remote device on Tailscale
tailscale status

# "relay" means passing through Tailscale relay servers
# No relay = direct peer-to-peer (best performance)
```

**Connection Types:**
- **Direct P2P:** Fastest (low latency, no relay)
- **Relay:** Through Tailscale servers (slightly higher latency)
- **Dead (Offline):** Device is offline

### Bandwidth Considerations

Tailscale uses about the same bandwidth as direct internet connection.

For video streaming (Jellyfin):
- 1080p: ~5 Mbps
- 4K: ~25 Mbps
- Ensure your internet connection supports this

## Troubleshooting

### Connection Refused

**Symptom:** Can't access services on Tailscale IP

```bash
# 1. Verify Tailscale is running on server
sudo systemctl status tailscale

# 2. Check IP address
tailscale ip -4

# 3. Test ping from remote device
ping 100.64.123.45

# 4. Check firewall not blocking ports
sudo ufw status

# 5. Verify service is running
docker ps | grep jellyfin
```

### Device Not Appearing

**Symptom:** Device not showing in Tailscale status

```bash
# 1. Restart Tailscale
sudo systemctl restart tailscale

# 2. Check logs
sudo journalctl -u tailscale -n 50

# 3. Re-authenticate
sudo tailscale up

# 4. Check internet connection
ping google.com
```

### Slow Connection (Using Relay)

**Symptom:** Connection going through relay instead of direct

Possible causes:
- Firewall blocking peer-to-peer connections
- NAT traversal issues
- Both devices on same network trying to use Tailscale

Solutions:
```bash
# 1. Check if devices are on same LAN
# If yes, use direct local IP instead

# 2. Configure firewall for UDP port 41641
sudo ufw allow 41641/udp

# 3. Restart Tailscale
sudo systemctl restart tailscale
```

### DNS Not Resolving

**Symptom:** Device names like `homeserver` don't resolve

```bash
# 1. Enable MagicDNS
sudo tailscale up --accept-dns=true

# 2. Verify in admin panel that DNS is configured
# https://login.tailscale.com/admin/dns

# 3. On remote devices, check DNS settings in OS

# 4. Try pinging IP directly instead
ping 100.64.123.45
```

### Can't Access Services

**Symptom:** Connected to Tailscale but services show "connection refused"

```bash
# 1. Verify services are running
docker compose ps

# 2. Check service is listening on all interfaces
sudo netstat -tlnp | grep 8096

# 3. Verify firewall isn't blocking port
sudo ufw allow 8096

# 4. Check compose config for localhost-only binding
# Change 127.0.0.1:8096 to 0.0.0.0:8096 or just :8096

# 5. Restart service
docker compose restart jellyfin
```

### Tailscale Won't Start

**Symptom:** Service fails to start or crashes

```bash
# 1. Check service status
sudo systemctl status tailscale

# 2. View logs
sudo journalctl -u tailscale -n 100

# 3. Try manual start for more error details
sudo /usr/sbin/tailscaled -state=/var/lib/tailscale/tailscaled.state

# 4. Restart and check again
sudo systemctl restart tailscale

# 5. If still failing, reinstall
sudo apt-get remove tailscale
sudo apt-get install tailscale
```

## Usage Scenarios

### Scenario 1: Watch Jellyfin While Traveling

1. Connect phone to Tailscale VPN
2. Open Jellyfin app or browser: `http://homeserver:8096`
3. Stream your media library securely

### Scenario 2: Manage Requests While Away

1. Connect to Tailscale from laptop
2. Access Jellyseerr: `http://homeserver:5055`
3. Approve or comment on content requests
4. Add new content to your library

### Scenario 3: Remote Troubleshooting

1. SSH to your server over Tailscale: `ssh user@homeserver`
2. Check logs, restart services
3. Or use Tailscale UI to view device details
4. No need to expose SSH to internet

### Scenario 4: Family Access

1. Create account for family member
2. Add their devices to Tailscale
3. They see your homeserver in their Tailscale devices
4. They can access Jellyfin securely (no public port)

## Migrating from Port Forwarding

If previously using port forwarding, migrate to Tailscale:

### Before (Port Forwarding):
- Exposed port 8096 to internet
- Complex firewall rules
- Security risk
- High latency on relays

### After (Tailscale):
- Only Tailscale network can see service
- Simple device management
- Secure encrypted tunnel
- Fast peer-to-peer connections

### Migration Steps:

1. **Add all devices to Tailscale**
   - Phone, laptop, tablets, etc.

2. **Verify services accessible via Tailscale**
   - Test all apps work via Tailscale IPs

3. **Disable port forwarding**
   - Remove rules from router
   - Remove firewall exceptions

4. **Remove UPnP settings** (if using)
   - In CasaOS or Docker settings

5. **Test external access**
   - Disconnect from home WiFi
   - Use mobile data
   - Confirm everything works via Tailscale

## Next Steps

After Tailscale is set up:

1. **Connect all your personal devices** to Tailscale
2. **Set device names** for easy reference
3. **Configure DNS** for MagicDNS (optional)
4. **Test each service** (Jellyfin, Sonarr, etc.)
5. **Share access** with family members (if desired)
6. **Remove port forwarding** from your router
7. **Update bookmarks** to use Tailscale IPs/names

## Resources

**Official Tailscale:**
- Website: https://tailscale.com
- Documentation: https://tailscale.com/kb
- Admin Panel: https://login.tailscale.com
- GitHub: https://github.com/tailscale/tailscale

**Community:**
- Discord: https://tailscale.com/discord
- GitHub Discussions: https://github.com/tailscale/tailscale/discussions
- Reddit: r/tailscale

## Pricing

**Tailscale Free (Most Homelab Users):**
- Up to 100 devices
- All features except advanced SSO
- Perfect for home & remote access

**Tailscale Business/Enterprise:**
- More devices, advanced admin controls
- For businesses (not needed for homelab)

Your homelab setup will work perfectly on the free plan.
