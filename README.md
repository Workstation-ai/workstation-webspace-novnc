# workstation-webspace-novnc

Remote desktop in your browser. For developers who need a quick GUI on a headless server.

**Works on:** Linux VMs, servers, bare metal, Docker, Codespaces.  
**Note:** Containers run in direct mode (no auto-restart on crash).

## Quick start

```bash
git clone -b v2-agent-environment https://github.com/Workstation-ai/workstation-webspace-novnc.git
cd workstation-webspace-novnc
cp .env.example .env
```

Edit `.env` with your password:

```bash
nano .env
```

Save and exit (Ctrl+X, Y, Enter), then:

```bash
./run
```

That's it. You get a URL and password.

## What it does

Sets up a virtual desktop accessible from any browser:
- Xvfb (virtual display) + fluxbox (window manager)
- noVNC (browser access) + cloudflare tunnel (public URL)
- xfce4-terminal + Firefox pre-installed
- Clipboard sharing between browser and desktop

## Commands

| Command | What |
|---------|------|
| `./run` | Install everything and start |
| `./run --agent-preset-workspace` | Install desktop + OpenCode GentleAI agent |
| `./stop` | Stop all services |
| `./status` | Show status + URL |
| `./uninstall` | Clean removal |

### Agent workspace

To also install the OpenCode GentleAI agent environment:

```bash
./run --agent-preset-workspace
```

This runs the [alpine-opencode-gentleai](https://github.com/reflecterlabs/alpine-opencode-gentleai) setup after the desktop is ready.

## Configuration (.env)

```bash
VNC_PASSWORD=yourpassword    # required
SCREEN_RESOLUTION=1920x1080  # optional
COLOR_DEPTH=24               # optional
```

## Usage on a new machine

```bash
git clone -b v2-agent-environment https://github.com/Workstation-ai/workstation-webspace-novnc.git
cd workstation-webspace-novnc
cp .env.example .env
nano .env
```

Set your `VNC_PASSWORD`, save (Ctrl+X, Y, Enter), then:

```bash
./run
```

## Notes

- Cloudflare quick tunnel URLs change on restart
- Desktop starts empty - right-click to open apps
- `Super+U` restores all minimized windows
- `pbcopy`/`pbpaste` work for clipboard sharing
