# noVNC Browser Use - Setup Learnings

## Architecture

The stack consists of 5 systemd services running as `bux` user:

1. **bux-xvfb** - Virtual framebuffer (Xvfb) on display :99, 1920x1080x24
2. **bux-fluxbox** - Lightweight window manager
3. **bux-x11vnc** - VNC server exposing display :99 on port 5900
4. **bux-novnc** - websockify translating WebSocket (port 6080) to VNC (port 5900)
5. **bux-novnc-tunnel** - cloudflare tunnel exposing port 6080 to the internet

## Key Learnings

### 1. AWS Security Groups Block Ports
- Even with iptables rules, AWS security groups control external access
- Port 6080 was blocked by default
- Solution: Use cloudflare tunnel (already installed on the box) to expose the service
- Cloudflare quick tunnels assign random URLs like `https://xxx.trycloudflare.com`
- These URLs change on restart - not suitable for production

### 2. Package Installation Gotchas
- `chromium-browser` on Ubuntu 24.04 is a snap transitional package
- Snap chromium works but has cgroup warnings when run from systemd
- `websockify` can't be installed via pip without `--break-system-packages` on Python 3.12+
- NoVNC must be cloned from GitHub (not available as apt package)

### 3. Xvfb Configuration
- `-screen 0 1920x1080x24` sets resolution to 1920x1080 with 24-bit color
- `-ac` disables access control (needed for VNC without auth)
- Keysym warnings are harmless (missing multimedia keys)

### 4. x11vnc Configuration
- `-forever` keeps listening after client disconnects
- `-shared` allows multiple VNC clients
- `-nopw` disables password (no auth - OK for cloudflare tunnel with HTTPS)
- `-noxdamage` fixes rendering issues with some X11 clients
- `-rfbport 5900` is the standard VNC port

### 5. noVNC / websockify
- websockify translates WebSocket connections to raw VNC
- `--web /opt/noVNC` serves the noVNC web client
- Port 6080 is the standard noVNC port
- The VNC connection URL is auto-detected by noVNC client

### 6. Chromium on Virtual Display
- Must use `--no-sandbox` when running as non-root
- `--disable-gpu` prevents GPU-related crashes on virtual display
- `--disable-dev-shm-usage` prevents /dev/shm issues in containers
- Launch with `DISPLAY=:99 /snap/bin/chromium [options]`

## Service Management

```bash
# Check status
~/novnc-browseruse/novnc-manager.sh status

# Get current tunnel URL
~/novnc-browseruse/novnc-manager.sh url

# Restart all services
~/novnc-browseruse/novnc-manager.sh restart
```

## Files

- `/etc/systemd/system/bux-xvfb.service`
- `/etc/systemd/system/bux-fluxbox.service`
- `/etc/systemd/system/bux-x11vnc.service`
- `/etc/systemd/system/bux-novnc.service`
- `/etc/systemd/system/bux-novnc-tunnel.service`
- `/opt/noVNC/` - noVNC web client
- `/var/log/bux/novnc-tunnel.log` - tunnel log with current URL
- `~/novnc-browseruse/novnc-manager.sh` - management script

## Security Notes

- VNC has no password (`-nopw`) - rely on cloudflare tunnel HTTPS
- Cloudflare quick tunnels have no uptime guarantee
- For production, use a named cloudflare tunnel with auth
- Consider adding VNC password for direct access

## Authentication Setup (Added Later)

### VNC Password
- Set via: `x11vnc -storepasswd bux2026 /home/bux/.vnc/passwd`
- Applied in service: `-rfbauth /home/bux/.vnc/passwd`

### Web Interface Auth (HTTP Basic)
- websockify 0.13.0 uses `--web-auth --auth-plugin --auth-source`
- `--auth-source` takes `username:password` directly, NOT a file
- Service config: `--auth-source admin:bux2026`
- Credentials: admin / bux2026

### Two Layers of Security
1. **Web UI**: HTTP Basic Auth (admin/bux2026) - prompts in browser
2. **VNC Protocol**: VNC password (bux2026) - prompts in VNC client

### DNS Resolution Note
- Local DNS (systemd-resolved) may not resolve cloudflare quick tunnel URLs immediately
- Works fine from external networks/browsers
- Use `curl --resolve` or external DNS (8.8.8.8) for local testing

## Critical Fix: Web Auth Blocks WebSocket

### Problem
`--web-auth` on websockify protects ALL paths including `/websockify` (the WebSocket endpoint). This prevents the noVNC client from establishing the VNC connection, resulting in directory listing or blank screen.

### Solution
Remove `--web-auth` from websockify. Use VNC password only:
- x11vnc uses `-rfbauth /home/bux/.vnc/passwd` for VNC protocol auth
- noVNC client shows a password dialog automatically when connecting
- This is the standard noVNC authentication method

### If Username+Password is Really Needed
Requires a reverse proxy (nginx) in front of websockify:
1. nginx handles HTTP Basic Auth on all HTTP requests
2. nginx proxies `/websockify` to websockify without auth
3. More complex but gives username+password on the web UI

### Current Setup
- VNC password: `bux2026` (prompted by noVNC client)
- No web-level auth (websockify serves files freely)
- VNC server uses `-rfbauth` for protocol-level authentication

## Critical: Snap Apps Don't Work from systemd

### Problem
Ubuntu 24.04 ships Firefox and Chromium ONLY as snaps. Snap apps check their cgroup and refuse to run from systemd services:
```
/system.slice/bux-apps.service is not a snap cgroup for tag snap.firefox.firefox
```

### Solution
Download Firefox directly from Mozilla as a tarball:
```bash
curl -sL "https://download.mozilla.org/?product=firefox-latest&os=linux64&lang=en-US" -o firefox.tar.bz2
cd /opt && tar xJf /tmp/firefox.tar.bz2
ln -sf /opt/firefox/firefox /usr/local/bin/firefox
```
This installs Firefox 152.0.3 as a native binary that works from systemd.

### Apps That Work from systemd
- xfce4-terminal ✓ (native apt package)
- Firefox ✓ (from Mozilla tarball)
- Chromium ✗ (snap only, won't work)

### Menu Configuration
Fluxbox menu at `~/.fluxbox/menu` with:
- Terminal (xfce4-terminal)
- Firefox (/opt/firefox/firefox)
- Text Editor (nano via terminal)
- System tools (htop, df, ps)
- Network info (curl ifconfig.me)

## Quick Tunnel URL Ephemeral Nature

### Problem
Each time `bux-novnc-tunnel` restarts, cloudflare assigns a new random URL. Previous URLs die immediately.

### Implications
- `Restart=always` in systemd means a crash generates a new URL
- Manual restarts (for config changes) also generate new URLs
- Users must rebookmark after every restart
- Not suitable for production - need a named cloudflare tunnel with account

### Management
```bash
# Get current URL without restarting
~/novnc-browseruse/novnc-manager.sh url
```

## Directory Listing Instead of noVNC UI

### Problem
Accessing `http://host:6080/` shows directory listing instead of the VNC interface.

### Solution
Create symlink: `ln -sf /opt/noVNC/vnc.html /opt/noVNC/index.html`
websockify's built-in web server doesn't have a default index option.

## Keyboard Input Stops Working

### Problem
After restarting fluxbox or x11vnc, keyboard input in noVNC stops working.

### Solution
Restart x11vnc service: `sudo systemctl restart bux-x11vnc`
The connection needs to be re-established after service restarts.

### Prevention
Avoid unnecessary restarts of x11vnc. It's stable and doesn't need periodic restarts.

## Fluxbox Menu Configuration

### File: `~/.fluxbox/menu`
```
[begin] (bux Desktop)
	[submenu] (Applications)
		[exec] (Terminal) {xfce4-terminal} <>
		[exec] (Firefox) {/opt/firefox/firefox} <>
		[exec] (Text Editor) {xfce4-terminal -e nano} <>
	[end]
	[submenu] (System)
		[exec] (htop) {xfce4-terminal -e htop} <>
		[exec] (Disk Usage) {xfce4-terminal -e "df -h"} <>
		[exec] (Process List) {xfce4-terminal -e "ps aux"} <>
	[end]
	[submenu] (Network)
		[exec] (IP Info) {xfce4-terminal -e "curl -s ifconfig.me && echo && ip addr show"} <>
	[end]
	[separator]
	[restart] (Restart) <>
	[exit] (Exit) <>
end
```

## Clipboard Sharing

### pbcopy/pbpaste Aliases
Added to `~/.bashrc`:
```bash
alias pbcopy='xsel --clipboard --input'
alias pbpaste='xsel --clipboard --output'
```
Clipboard shares between noVNC browser and virtual desktop via X11 PRIMARY selection.

## Service Startup Order

1. `bux-xvfb` (display server)
2. `bux-fluxbox` (window manager, needs display)
3. `bux-apps` (terminal + Firefox, needs window manager)
4. `bux-x11vnc` (VNC server, needs display)
5. `bux-novnc` (websocket proxy, needs VNC)
6. `bux-novnc-tunnel` (cloudflare, needs websocket proxy)
