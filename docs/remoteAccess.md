# Remote Desktop Access for Ubuntu 22.04 Desktop Server

## Overview

This document provides a comprehensive guide for setting up remote desktop access to an Ubuntu 22.04 Desktop server, allowing Windows and macOS clients to connect and view the **same physical desktop session** that is displayed on the server's monitor. This approach ensures that applications left running on the physical desktop remain accessible when connecting remotely.

## Table of Contents

1. [Understanding the Requirements](#understanding-the-requirements)
2. [Solution Overview](#solution-overview)
3. [Prerequisites](#prerequisites)
4. [Server Setup](#server-setup)
5. [Client Configuration](#client-configuration)
6. [Security Hardening](#security-hardening)
7. [Troubleshooting](#troubleshooting)
8. [Headless Access (No Monitor Connected)](#headless-access-no-monitor-connected)
9. [Alternative Solutions](#alternative-solutions)

---

## Understanding the Requirements

### Key Requirement: Session Persistence

The critical requirement is to **share the existing physical desktop session** rather than creating a new virtual desktop. This means:

- ✅ Applications running on the physical monitor remain visible when connecting remotely
- ✅ Remote users see exactly what is displayed on the physical screen
- ✅ Both local and remote users can interact with the same desktop simultaneously
- ✅ Logging out remotely does not terminate applications running on the physical desktop

### Why This Matters

Many VNC server implementations (like TigerVNC's default `Xvnc` or `vino`) create separate virtual desktop sessions. This approach would **not** meet the requirement because:

- Applications running on the physical desktop would not be visible remotely
- You would have two separate desktop environments
- Switching between them would require logging out and back in

---

## Solution Overview

### Recommended Solution: x11vnc

**x11vnc** is the optimal solution for this use case because it is specifically designed to share an existing X11 display session. It mirrors the physical desktop exactly as it appears on the monitor.

#### Key Advantages

- ✅ **Direct Physical Desktop Sharing**: Shares the exact desktop session displayed on the physical monitor
- ✅ **Cross-Platform Client Support**: Compatible with RealVNC Viewer and other standard VNC clients on Windows, macOS, and Linux
- ✅ **Session Persistence**: Applications remain running and visible whether accessed locally or remotely
- ✅ **Simultaneous Access**: Multiple users can view and control the same desktop concurrently
- ✅ **Mature and Stable**: Well-established solution with extensive documentation

#### Important Limitation: X11 Required

> [!IMPORTANT]
> x11vnc **only works with X11 (Xorg)** and is **incompatible with Wayland**. Ubuntu 22.04 uses Wayland by default, so you must switch to X11 for x11vnc to function.

---

## Prerequisites

### System Requirements

- Ubuntu 22.04 Desktop (or later)
- Physical access to the server for initial setup
- Network connectivity between server and client machines
- Administrative (sudo) privileges

### Display Server: X11 vs Wayland

Ubuntu 22.04 uses **Wayland** as the default display server, but x11vnc requires **X11 (Xorg)**. You must verify and potentially switch your display server.

#### Check Current Display Server

```bash
echo $XDG_SESSION_TYPE
```

- If output is `wayland`, you need to switch to X11
- If output is `x11`, you're already using X11

---

## Server Setup

### Step 1: Switch from Wayland to X11

Choose one of the following methods:

#### Method A: Switch at Login (Recommended for Single User)

1. **Log out** of your current session
2. At the login screen, **before entering your password**:
   - Click the **gear icon** (⚙️) in the bottom-right corner
   - Select **"Ubuntu on Xorg"** or **"GNOME on Xorg"**
3. Log in with your credentials
4. This setting will persist for your user account across reboots

#### Method B: Disable Wayland System-Wide (GDM)

If you're using GDM (GNOME Display Manager), you can disable Wayland for all users:

```bash
sudo nano /etc/gdm3/custom.conf
```

Uncomment the following line (remove the `#`):

```ini
WaylandEnable=false
```

Save the file (`Ctrl+O`, `Enter`, `Ctrl+X`) and reboot:

```bash
sudo reboot
```

#### Method C: Install LightDM (Alternative Display Manager)

Some users prefer LightDM as it's simpler and works well with x11vnc:

```bash
sudo apt update
sudo apt install lightdm
```

During installation, select **LightDM** as the default display manager when prompted. Then reboot:

```bash
sudo reboot
```

#### Verify X11 is Active

After switching, verify you're using X11:

```bash
echo $XDG_SESSION_TYPE
```

Expected output: `x11`

---

### Step 2: Install x11vnc

Install the x11vnc package:

```bash
sudo apt update
sudo apt install x11vnc
```

---

### Step 3: Set a VNC Password

Create a password file for secure VNC connections:

```bash
x11vnc -storepasswd
```

You'll be prompted to:
1. Enter a password (8 characters recommended)
2. Confirm the password
3. Optionally set a view-only password (recommended: choose `n` for no)

The password will be stored in `~/.vnc/passwd`

> [!WARNING]
> **Security Note**: This password is stored in an obfuscated format, not encrypted. For production environments, use SSH tunneling (see [Security Hardening](#security-hardening)).

---

### Step 4: Test x11vnc Manually

Before creating a systemd service, test x11vnc manually:

```bash
x11vnc -display :0 -auth guess -rfbauth ~/.vnc/passwd
```

**Explanation of flags:**
- `-display :0` - Share the primary display (physical monitor)
- `-auth guess` - Automatically find the X authority file
- `-rfbauth ~/.vnc/passwd` - Use the password file created earlier

You should see output indicating x11vnc is running and listening on port 5900.

**Test the connection** from a client machine using RealVNC Viewer or another VNC client:
- Connect to: `<server-ip-address>:5900` or `<server-ip-address>:0`
- Enter the password you created

If successful, press `Ctrl+C` in the terminal to stop x11vnc and proceed to create a systemd service.

---

### Step 5: Create a Systemd Service for Auto-Start

To ensure x11vnc starts automatically when you log in, create a systemd user service.

#### Create the Service File

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/x11vnc.service
```

#### Add the Following Content

```ini
[Unit]
Description=x11vnc VNC Server for Display :0
After=graphical-session.target

[Service]
Type=simple
Environment=DISPLAY=:0
Environment=XAUTHORITY=/run/user/1000/gdm/Xauthority
ExecStart=/usr/bin/x11vnc -display :0 -auth guess -rfbauth %h/.vnc/passwd -forever -loop -noxdamage -repeat -shared
Restart=on-failure
RestartSec=3

[Install]
WantedBy=default.target
```

**Explanation of flags:**
- `-forever` - Keep running after client disconnects
- `-loop` - Restart if x11vnc crashes
- `-noxdamage` - Improves performance on some systems
- `-repeat` - Allow key repeat for better typing experience
- `-shared` - Allow multiple simultaneous connections
- `%h` - Expands to your home directory

> [!NOTE]
> **User ID Note**: The `XAUTHORITY` path includes `/run/user/1000/`. The `1000` is typically the first user's UID. If you're not the first user, find your UID with `id -u` and adjust accordingly.

Save the file (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Step 6: Enable and Start the Service

Enable the service to start automatically on login:

```bash
systemctl --user enable x11vnc.service
systemctl --user start x11vnc.service
```

#### Verify the Service is Running

```bash
systemctl --user status x11vnc.service
```

You should see `active (running)` in green.

#### Check x11vnc is Listening

```bash
ss -tuln | grep 5900
```

Expected output should show port 5900 in LISTEN state.

---

### Step 7: Configure Firewall (If Enabled)

If you have UFW (Uncomplicated Firewall) enabled, allow VNC connections:

```bash
sudo ufw allow 5900/tcp
sudo ufw status
```

> [!CAUTION]
> **Security Warning**: Opening port 5900 to your entire network exposes your desktop to anyone who knows the password. See [Security Hardening](#security-hardening) for safer alternatives.

---

## Client Configuration

### Windows Clients

#### Option 1: RealVNC Viewer (Recommended)

1. **Download RealVNC Viewer**:
   - Visit: https://www.realvnc.com/en/connect/download/viewer/
   - Download the Windows installer
   - Install the application

2. **Connect to the Server**:
   - Open RealVNC Viewer
   - Enter the server address: `<server-ip-address>:5900` or `<server-ip-address>`
   - Click **Connect**
   - Enter the VNC password when prompted
   - Accept the security warning (if using unencrypted connection)

#### Option 2: TightVNC Viewer

1. Download from: https://www.tightvnc.com/download.php
2. Install and connect using the same server address format

---

### macOS Clients

#### Option 1: RealVNC Viewer (Recommended)

1. **Download RealVNC Viewer**:
   - Visit: https://www.realvnc.com/en/connect/download/viewer/
   - Download the macOS installer
   - Install the application

2. **Connect to the Server**:
   - Same process as Windows (see above)

#### Option 2: Built-in Screen Sharing (macOS)

macOS includes a built-in VNC client:

1. Open **Finder**
2. Press `Cmd+K` (or Go → Connect to Server)
3. Enter: `vnc://<server-ip-address>:5900`
4. Click **Connect**
5. Enter the VNC password when prompted

---

### Finding Your Server IP Address

On the Ubuntu server, run:

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

Look for an address like `192.168.1.100` or `10.0.0.50` (your local network IP).

---

## Security Hardening

> [!CAUTION]
> **Critical Security Warning**: VNC traffic is **not encrypted by default**. Anyone on your network can potentially intercept your password and desktop session. The following security measures are **strongly recommended** for production use.

### Option 1: SSH Tunnel (Recommended)

SSH tunneling encrypts all VNC traffic and eliminates the need to expose port 5900.

#### Server Setup

1. **Ensure SSH server is installed and running**:

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

2. **Modify x11vnc to listen only on localhost**:

Edit your systemd service file:

```bash
nano ~/.config/systemd/user/x11vnc.service
```

Change the `ExecStart` line to include `-localhost`:

```ini
ExecStart=/usr/bin/x11vnc -localhost -display :0 -auth guess -rfbauth %h/.vnc/passwd -forever -loop -noxdamage -repeat -shared
```

Reload and restart the service:

```bash
systemctl --user daemon-reload
systemctl --user restart x11vnc.service
```

3. **Close VNC port in firewall** (if previously opened):

```bash
sudo ufw delete allow 5900/tcp
```

#### Client Setup (Windows)

1. **Install an SSH client** (Windows 10/11 includes OpenSSH by default)

2. **Create SSH tunnel** before connecting with VNC:

Open PowerShell or Command Prompt:

```powershell
ssh -L 5900:localhost:5900 username@server-ip-address
```

Replace `username` with your Ubuntu username and `server-ip-address` with your server's IP.

3. **Keep the SSH session open** and connect VNC client to:
   - Address: `localhost:5900` or `127.0.0.1:5900`

#### Client Setup (macOS/Linux)

Same as Windows, using Terminal:

```bash
ssh -L 5900:localhost:5900 username@server-ip-address
```

Then connect VNC client to `localhost:5900`.

---

### Option 2: VPN

Use a VPN solution (like WireGuard, OpenVPN, or Tailscale) to create a secure network tunnel. This approach:
- Encrypts all traffic between client and server
- Provides network-level security
- Allows access from outside your local network securely

**Tailscale** (easiest option):
1. Install Tailscale on both server and clients: https://tailscale.com/download
2. Connect both devices to your Tailscale network
3. Use the Tailscale IP address to connect via VNC

---

### Option 3: TLS Encryption (Advanced)

x11vnc supports built-in TLS encryption, but requires certificate management and is more complex to set up. SSH tunneling is generally preferred.

---

### Additional Security Best Practices

1. **Use Strong Passwords**: VNC passwords should be complex and unique
2. **Limit Access by IP**: Use firewall rules to restrict VNC access to specific IP addresses
3. **Disable Root Login**: Never allow VNC access as the root user
4. **Regular Updates**: Keep x11vnc and Ubuntu updated:
   ```bash
   sudo apt update && sudo apt upgrade
   ```
5. **Monitor Logs**: Check for unauthorized access attempts:
   ```bash
   journalctl --user -u x11vnc.service
   ```
6. **Use View-Only Mode**: For monitoring purposes, create a view-only password:
   ```bash
   x11vnc -storepasswd /path/to/viewonly.passwd
   ```

---

## Troubleshooting

### Issue: "Cannot open display :0"

**Cause**: x11vnc cannot find the X display.

**Solutions**:

1. **Verify you're using X11** (not Wayland):
   ```bash
   echo $XDG_SESSION_TYPE
   ```
   Should output `x11`.

2. **Check DISPLAY variable**:
   ```bash
   echo $DISPLAY
   ```
   Should output `:0` or `:1`.

3. **Find the correct X authority file**:
   ```bash
   x11vnc -findauth
   ```
   Use the path shown in your systemd service file.

---

### Issue: "Connection Refused" from Client

**Cause**: x11vnc is not running or firewall is blocking connections.

**Solutions**:

1. **Check if x11vnc is running**:
   ```bash
   systemctl --user status x11vnc.service
   ss -tuln | grep 5900
   ```

2. **Check firewall rules**:
   ```bash
   sudo ufw status
   ```
   Ensure port 5900 is allowed (if not using SSH tunnel).

3. **Verify server IP address**:
   ```bash
   ip addr show
   ```
   Ensure you're using the correct IP on the client.

---

### Issue: Black Screen or Frozen Display

**Cause**: Display driver or X11 damage extension issues.

**Solutions**:

1. **Add `-noxdamage` flag** (already included in the service file above)

2. **Try alternative flags**:
   Edit the systemd service and add `-noxrecord`:
   ```ini
   ExecStart=/usr/bin/x11vnc -display :0 -auth guess -rfbauth %h/.vnc/passwd -forever -loop -noxdamage -noxrecord -repeat -shared
   ```

3. **Restart the service**:
   ```bash
   systemctl --user daemon-reload
   systemctl --user restart x11vnc.service
   ```

---

### Issue: Keyboard Input Not Working

**Cause**: Key repeat or input method issues.

**Solution**:

Ensure `-repeat` flag is included in the x11vnc command (already in the service file above).

---

### Issue: Service Doesn't Start on Boot

**Cause**: Systemd user service timing issues.

**Solutions**:

1. **Enable lingering** (allows user services to run without active login):
   ```bash
   loginctl enable-linger $USER
   ```

2. **Check service logs**:
   ```bash
   journalctl --user -u x11vnc.service -b
   ```

3. **Verify service is enabled**:
   ```bash
   systemctl --user is-enabled x11vnc.service
   ```
   Should output `enabled`.

---

### Issue: Multiple Monitors Not Showing

**Cause**: x11vnc may only share the primary display.

**Solution**:

x11vnc shares all monitors in your X session by default. If you're only seeing one monitor:

1. **Check your display configuration**:
   ```bash
   xrandr
   ```

2. **Ensure monitors are configured as a single X screen** (not separate screens)

3. **For multi-monitor setups**, you may need to adjust the `-clip` option or use `-multiptr` for better support

---

### Issue: Poor Performance or Lag

**Cause**: Network bandwidth or encoding issues.

**Solutions**:

1. **Adjust VNC client quality settings**: Most VNC clients allow you to reduce color depth or quality for better performance

2. **Add compression flags** to x11vnc:
   ```ini
   ExecStart=/usr/bin/x11vnc -display :0 -auth guess -rfbauth %h/.vnc/passwd -forever -loop -noxdamage -repeat -shared -scale 3/4
   ```
   The `-scale 3/4` reduces resolution to 75% for faster transmission.

3. **Use a wired connection** instead of Wi-Fi when possible

---

## Headless Access (No Monitor Connected)

When no physical monitor is connected to the server, the display server may not initialize a graphical session, causing remote desktop connections to fail. This section covers solutions for accessing Ubuntu desktop remotely without a physical monitor.

### Option 1: Hardware Dummy Plug (Recommended)

The simplest and most reliable solution is a **dummy HDMI/DisplayPort plug** (also called an HDMI emulator or headless ghost adapter). This small device plugs into your GPU's video output and emulates a connected monitor.

**Advantages**:
- Works with both X11 and Wayland
- No configuration changes required
- Supports high resolutions (1080p, 4K depending on adapter)
- Inexpensive (~$5-15 USD)

**How to use**:
1. Purchase a dummy plug compatible with your video output (HDMI, DisplayPort, DVI, or VGA)
2. Plug it into an available video port on your server
3. The system will detect it as a real monitor
4. Remote desktop will work normally

**Search terms**: "HDMI dummy plug", "headless HDMI adapter", "HDMI ghost display emulator"

---

### Option 2: X11 Dummy Video Driver (Software Solution)

If a hardware dummy plug is not available, you can configure X.org to use a **dummy video driver** that software-emulates a monitor. This approach requires X11 (will not work with Wayland).

> [!WARNING]
> This method only works with X11. If you need Wayland, use a hardware dummy plug or see [Option 3](#option-3-wayland-with-gnome-remote-desktop).

#### Step 1: Install the Dummy Driver

```bash
sudo apt install xserver-xorg-video-dummy
```

#### Step 2: Create an Xorg Configuration File

Create a new configuration file:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/10-dummy.conf
```

#### Step 3: Add the Dummy Configuration

Add the following content, adjusting the resolution as needed (e.g., `1920x1080`, `2560x1440`):

```
Section "Device"
    Identifier "Configured Video Device"
    Driver "dummy"
EndSection

Section "Monitor"
    Identifier "Configured Monitor"
    HorizSync 31.5-48.5
    VertRefresh 50-70
EndSection

Section "Screen"
    Identifier "DefaultScreen"
    Device "Configured Video Device"
    Monitor "Configured Monitor"
    DefaultDepth 24
    SubSection "Display"
        Depth 24
        Modes "1920x1080"
    EndSubSection
EndSection
```

Save the file (`Ctrl+O`, `Enter`, `Ctrl+X`).

#### Step 4: Ensure X11 is Configured

The dummy driver requires X11. If not already done, disable Wayland:

```bash
sudo nano /etc/gdm3/custom.conf
```

Uncomment or add:

```ini
WaylandEnable=false
```

#### Step 5: Restart the Display Manager

```bash
sudo systemctl restart gdm3
```

The system will now load the dummy driver and create a virtual display, allowing x11vnc to share the graphical session even without a physical monitor.

> [!NOTE]
> **Reverting**: To disable the dummy driver (e.g., when reconnecting a real monitor), either delete or rename the configuration file:
> ```bash
> sudo mv /etc/X11/xorg.conf.d/10-dummy.conf /etc/X11/xorg.conf.d/10-dummy.conf.disabled
> sudo systemctl restart gdm3
> ```

---

### Option 3: Wayland with GNOME Remote Desktop

If you prefer to keep **Wayland** as your display server, Ubuntu 22.04+ includes **GNOME Remote Desktop** which provides built-in screen sharing via the RDP protocol. This works natively with Wayland.

> [!IMPORTANT]
> GNOME Remote Desktop typically requires an active graphical session. For fully headless Wayland setups, a hardware dummy plug is still recommended.

#### Enable GNOME Remote Desktop

1. Open **Settings** → **Sharing**
2. Enable **Remote Desktop**
3. Configure authentication (set a username/password or use your system credentials)
4. Note the connection address shown

#### Connect from Windows

Windows has a built-in RDP client:

1. Press `Win+R`, type `mstsc`, press Enter
2. Enter the Ubuntu server's IP address
3. Click **Connect** and enter credentials

#### Connect from macOS

1. Download **Microsoft Remote Desktop** from the App Store
2. Add a new PC with the Ubuntu server's IP address
3. Connect and enter credentials

#### Limitations with Headless Wayland

- GNOME Remote Desktop may not start properly without a display
- For reliable headless Wayland access, combine with a hardware dummy plug
- Alternatively, consider using X11 with the dummy driver for headless scenarios

---

### Comparison: Headless Solutions

| Solution | Display Server | Complexity | Cost | Reliability |
|----------|---------------|------------|------|-------------|
| Hardware Dummy Plug | X11 or Wayland | Low | ~$10 | High |
| X11 Dummy Driver | X11 only | Medium | Free | Medium |
| GNOME Remote Desktop | Wayland | Low | Free | Varies* |

*GNOME Remote Desktop reliability for headless use depends on Ubuntu version and system configuration.

---

## Alternative Solutions

While x11vnc is the recommended solution for sharing a physical desktop session, here are alternatives:

### 1. TigerVNC with x0vncserver

**Description**: TigerVNC's `x0vncserver` can also share an existing X display.

**Pros**:
- High performance
- Active development
- Good encoding options

**Cons**:
- More complex setup for physical desktop sharing
- Less documentation for this specific use case

**Basic Setup**:
```bash
sudo apt install tigervnc-standalone-server tigervnc-common
x0vncserver -display :0 -passwordfile ~/.vnc/passwd
```

---

### 2. RealVNC Server (Commercial)

**Description**: RealVNC offers a commercial server product with desktop sharing capabilities.

**Pros**:
- Professional support
- Cloud connectivity options
- Cross-platform consistency
- Built-in encryption

**Cons**:
- Requires paid license for full features
- May be overkill for simple local network use

**Website**: https://www.realvnc.com/

---

### 3. NoMachine

**Description**: A proprietary remote desktop solution with free and commercial versions.

**Pros**:
- Excellent performance
- Built-in encryption
- File transfer capabilities
- Free for personal use

**Cons**:
- Proprietary protocol (not standard VNC)
- Requires NoMachine client on all devices

**Website**: https://www.nomachine.com/

---

### 4. Chrome Remote Desktop

**Description**: Google's remote desktop solution.

**Pros**:
- Easy setup
- Works through firewalls (cloud-based)
- Free
- Cross-platform

**Cons**:
- Requires Google account
- Cloud-based (privacy considerations)
- May not share physical desktop session reliably

---

### 5. RDP with xrdp (Not Recommended for This Use Case)

**Description**: xrdp provides RDP protocol support on Linux.

**Cons for this use case**:
- Creates a **separate virtual session**, not sharing the physical desktop
- Does not meet the requirement of seeing applications left open on the physical screen

**Only use if**: You want separate sessions for local and remote access.

---

## Summary

This guide provides a complete solution for sharing your Ubuntu 22.04 Desktop's physical display with remote Windows and macOS clients using x11vnc and standard VNC clients like RealVNC Viewer.

### Quick Reference

**Server Setup Checklist**:
- [ ] Switch from Wayland to X11
- [ ] Install x11vnc: `sudo apt install x11vnc`
- [ ] Create password: `x11vnc -storepasswd`
- [ ] Create systemd service
- [ ] Enable and start service
- [ ] Configure firewall or SSH tunnel

**Client Connection**:
- Install RealVNC Viewer or use built-in VNC client
- Connect to `<server-ip>:5900`
- Enter VNC password

**Security**:
- Use SSH tunnel for encryption (recommended)
- Or use VPN for network-level security
- Never expose VNC directly to the internet

### Getting Help

- **x11vnc documentation**: `man x11vnc`
- **x11vnc GitHub**: https://github.com/LibVNC/x11vnc
- **Ubuntu Forums**: https://ubuntuforums.org/
- **Ask Ubuntu**: https://askubuntu.com/

---

**Document Version**: 1.1  
**Last Updated**: December 2025  
**Tested On**: Ubuntu 22.04 LTS Desktop

### Changelog

- **v1.1**: Added Headless Access section covering hardware dummy plugs, X11 dummy driver configuration, and Wayland with GNOME Remote Desktop
- **v1.0**: Initial release with x11vnc setup guide
