---
name: workstation-webspace-novnc
description: Browser-accessible remote desktop for headless servers
triggers:
  - noVNC
  - remote desktop
  - virtual display
  - VNC
  - web desktop
  - headless GUI
---

# workstation-webspace-novnc

One-command remote desktop for headless servers.

## Setup

```bash
cp .env.example .env
# edit .env with your password
./run
```

## Commands

- `./run` - Install and start
- `./stop` - Stop services
- `./status` - Show URL and status
- `./uninstall` - Clean removal

## .env

```bash
VNC_PASSWORD=changeme
SCREEN_RESOLUTION=1920x1080
COLOR_DEPTH=24
```

## What it provides

- Virtual display (Xvfb)
- Window manager (fluxbox) with right-click menu
- VNC server with password auth
- Web access via noVNC + cloudflare tunnel
- Clipboard sharing (pbcopy/pbpaste)
- Keyboard shortcut: Super+U = unminimize all
