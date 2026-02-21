Install Ubuntu on an old server / laptop

Overview
- This guide explains how to install Ubuntu (Server or Desktop) on older hardware and how to SSH into it from another machine.
- It covers preparing a bootable USB from Windows, BIOS/UEFI considerations, installation steps, basic SSH setup, key-based login, and troubleshooting.

Prerequisites
- A target machine (old server or laptop) with a working USB port.
- A second machine to create the USB installer (Windows, macOS, or Linux).
- A USB flash drive (4 GB+ for Server, 8 GB+ recommended for Desktop).
- Internet access for downloads and package updates.

1) Download Ubuntu
- Choose an image:
  - Ubuntu Server (minimal, headless)(Recommended): https://ubuntu.com/download/server
  - Ubuntu Desktop (graphical): https://ubuntu.com/download/desktop
- Download the appropriate ISO for your CPU (amd64 for most older PCs).

2) Make a bootable USB (Windows)
- Recommended tools: Rufus or balenaEtcher.
- Rufus steps (Windows):

```powershell
# 1. Insert USB
# 2. Open Rufus, select the downloaded ISO
# 3. Set Partition scheme: MBR for legacy BIOS, GPT for UEFI (choose based on your target machine)
# 4. File system: FAT32 (default)
# 5. Click Start and let Rufus write the USB
```

3) BIOS / UEFI settings on target machine
- Reboot and enter BIOS/UEFI (common keys: F2, F10, F12, DEL, ESC).
- Set USB as first boot device or use one-time boot menu (F12/F11) to boot from USB.
- If the machine supports Secure Boot and you use the Desktop ISO, Secure Boot is usually fine; for very old hardware there may be no Secure Boot.
- If using Legacy BIOS mode, ensure USB boots in legacy mode and choose MBR when creating USB.

4) Install Ubuntu
- Boot from the USB and follow the installer prompts.
- For server use the Server installer (includes option to install OpenSSH during install).
- Typical choices:
  - Language, keyboard
  - Network: usually DHCP; you can set a static IP if this is a server
  - Storage: "Use entire disk" unless you want custom partitions
  - Create a user: note the username (you'll SSH with it)
  - Optional: enable OpenSSH server when prompted (Server installer shows this option)

5) After installation first boot
- Log in on the machine or via a local keyboard.
- Update packages:

```bash
sudo apt update && sudo apt upgrade -y
```

6) Install/verify SSH server (if you didn't install it during setup)

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

- Allow SSH through the firewall (if using `ufw`):

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

7) Find the machine's IP address

```bash
# prints one or more IPs assigned to the host
hostname -I
# or show detailed info
ip addr show
```

8) SSH from another machine
- From Linux/macOS/Windows (PowerShell 7+ or Windows 10/11 builtin OpenSSH):

```bash
ssh username@192.168.x.y
```

- Replace `username` and `192.168.x.y` with the account and IP from the previous step.
- If the SSH server uses a non-standard port (e.g., 2222):

```bash
ssh -p 2222 username@192.168.x.y
```

9) Set up key-based authentication (recommended)
- Generate a key pair on the client (if you don't have one):

```bash
# on your client machine
ssh-keygen -t ed25519 -C "your_email@example.com"
```

- Copy the public key to the server:

```bash
# preferred if ssh-copy-id available
ssh-copy-id username@192.168.x.y

# or manual method from client
cat ~/.ssh/id_ed25519.pub | ssh username@192.168.x.y "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

- Test login (should not prompt for a password):

```bash
ssh username@192.168.x.y
```

- (Optional) Disable password auth on the server for stronger security. Edit `/etc/ssh/sshd_config` and set:

```
PasswordAuthentication no
PermitRootLogin no
```

Then reload SSH:

```bash
sudo systemctl reload ssh
```

10) Troubleshooting
- "Connection refused": ensure `openssh-server` installed and `ssh` service running.
- "Timed out": check networking (cables, Wi‑Fi), ensure IP is correct and firewall allows SSH.
- If using Wi‑Fi during install, older installers may require temporary wired connection.
- Use `sudo journalctl -u ssh -e` to inspect SSH logs on the server.

11) Security & maintenance tips
- Keep system updated regularly: `sudo apt update && sudo apt upgrade -y`.
- Use key-based auth and disable passwords.
- Consider installing `fail2ban` to reduce brute-force attempts.
- If exposing SSH to the internet, use a non-standard port, strong keys, and use VPN/Jump host where possible.

If you want, I can:
- Add instructions for a static IP configuration example.
- Show how to SSH from Windows with PuTTY or PowerShell step-by-step.
- Walk through enabling remote access over IPv6 or through a router (port forwarding).

