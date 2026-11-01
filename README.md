# rpi-wifi-config

A lightweight web interface for managing Wi-Fi connections on a Raspberry Pi running Raspberry Pi OS Bullseye or Bookworm (Debian 11/12) with NetworkManager.

Runs as a Docker container. Communicates with NetworkManager over D-Bus. Does not require any privileged container flags.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Docker](#docker)
- [Uninstalling](#uninstalling)
- [How it works](#how-it-works)
- [Security notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [API](#api)
- [CI/CD](#cicd)
- [File structure](#file-structure)

---

## Quick Start

```bash
git clone https://github.com/id0ntplaygam3s/rpi-wifi-config.git
cd rpi-wifi-config
bash install.sh
```

That's it. The installer checks your Pi, sets up NetworkManager, generates credentials, installs the polkit rule, and starts the container. Access the UI at `http://<PI_IP>:8090`.

---

## Features

- Current Wi-Fi status: SSID, signal strength, frequency, IP address
- Scan for nearby networks
- Connect to a network or update credentials for a saved one
- Add a new network manually (including hidden SSIDs)
- Remove a saved connection
- Automatic rollback if a connection attempt fails — ethernet is never affected

---

## Requirements

- Raspberry Pi OS Bullseye or Bookworm (Debian 11/12)
- Docker installed and your user in the `docker` group
- At least one Wi-Fi interface present (e.g. `wlan0` or `wlan1`) and not rfkill-blocked

To verify:

```bash
cat /etc/os-release
ip -o link show | grep -E '^[0-9]+: wl'
rfkill list wifi
```

On Bullseye, the installer will automatically install/enable NetworkManager and disable dhcpcd.

---

## Installation

```bash
git clone https://github.com/id0ntplaygam3s/rpi-wifi-config.git
cd rpi-wifi-config
bash install.sh
```

The installer will:

1. Check system compatibility (OS, Wi-Fi, Docker, polkit, D-Bus)
2. Ask whether to pull the pre-built image from Docker Hub (recommended) or build from source locally
3. Set up NetworkManager and disable dhcpcd conflicts
4. Prompt for web UI credentials (saved to `.env`, chmod 600)
5. Install the polkit rule (substituting your username)
6. Fix permissions on existing Wi-Fi connection profiles
7. Build/pull the Docker image and start the container

The installer is idempotent — re-running it updates the image and preserves your credentials.

---

## Docker

The installer pulls the pre-built image from Docker Hub by default. You can choose to build from source locally during installation.

```bash
docker pull id0ntplaygam3s/rpi-wifi-config:latest
```

The image is built for both `arm/v7` and `arm64` via GitHub Actions.

To manage the container manually:

```bash
docker compose up -d      # Start
docker compose down       # Stop
docker compose logs -f    # View logs
```

To build locally without the installer:

```bash
BUILD_LOCAL=true docker compose build
BUILD_LOCAL=true docker compose up -d
```

---

## Uninstalling

```bash
bash uninstall.sh
```

Stops the container, removes the polkit rule, and removes `.env`. The repo is left intact and ready to reinstall.

To fully remove everything:

```bash
cd .. && rm -rf rpi-wifi-config/
```

---

## How it works

The app is a Flask server running inside a Docker container as uid 1000 (non-root). It communicates with NetworkManager on the host via the D-Bus system socket, which is bind-mounted into the container at `/run/dbus/system_bus_socket`.

A polkit rule grants the Pi user permission to call NetworkManager D-Bus methods without a password prompt. This is the only host-level privilege required — no `--privileged` flag, no `sudo` inside the container.

When a connection attempt is made, the app snapshots the current active connection first. If the new connection fails to activate within 30 seconds, it automatically reactivates the previous one.

---

## Security notes

- Web UI is protected by HTTP Basic Auth
- Wi-Fi passphrases are passed directly to NetworkManager over D-Bus and are never logged
- The container runs as uid 1000 (non-root)
- No `--privileged` flag
- Only the D-Bus socket is mounted from the host
- The app never touches eth0 or any non-wireless connection
- Only one connection attempt can run at a time (threading lock)
- Port 8090 is bound to `0.0.0.0` — reachable on your LAN. Do not forward this port through your router.

---

## Troubleshooting

**"D-Bus error: AccessDenied" on connect/update**

The existing connection profile is not owned by the pi user. Run:

```bash
sudo nmcli connection modify "<connection name>" connection.permissions "user:pi"
```

The installer does this automatically for connections that exist at install time. This only occurs for connections added afterwards via another method.

**"WIFI_PASS environment variable not set" (503 error)**

The `.env` file is missing or `WIFI_PASS` is empty:

```bash
cat ~/rpi-wifi-config/.env
cd ~/rpi-wifi-config && docker compose up -d
```

**Scan returns no results**

NetworkManager rate-limits scan requests. Wait 30 seconds and try again.

**Connection attempt always fails / rolls back**

- Verify the passphrase is correct by testing on another device first
- Check logs: `docker compose logs wifi-config`
- Check Wi-Fi is not rfkill-blocked: `rfkill list wifi`

**App not reachable after Pi reboot**

The container is set to `restart: unless-stopped` and should start automatically:

```bash
docker compose ps
docker compose up -d
```

---

## API

All endpoints require Basic Auth except `/health`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/status` | Current Wi-Fi interface status |
| GET | `/api/scan` | Scan for nearby networks |
| GET | `/api/connections` | List saved Wi-Fi connections |
| POST | `/api/connect` | Connect or update a network |
| DELETE | `/api/connections/<uuid>` | Remove a saved connection |
| GET | `/health` | Health check (no auth) |

**POST /api/connect**

```json
{ "ssid": "MyNetwork", "passphrase": "mypassword" }
```

Leave `passphrase` empty or omit it for open networks.

---

## CI/CD

This project uses GitHub Actions to automatically build and push multi-architecture Docker images (ARMv7 + ARM64) to Docker Hub on every push to `main`.

To set up Docker Hub authentication for CI:

1. Generate an access token at [Docker Hub → Settings → Security](https://hub.docker.com/settings/security)
2. Add the token as a GitHub secret named `DOCKERHUB_TOKEN`
3. Add your Docker Hub username as a GitHub secret named `DOCKERHUB_USERNAME`

---

## File structure

```
rpi-wifi-config/
├── .github/workflows/
│   └── docker-publish.yml      # CI: builds & pushes to Docker Hub
├── install.sh                  # Run to install or update
├── uninstall.sh                # Run to remove
├── README.md
├── docker-compose.yml          # Uses pre-built image from Docker Hub
├── .env                        # Created by installer (not in git)
├── app/
│   ├── Dockerfile              # Used for building the image (CI)
│   ├── .dockerignore
│   ├── requirements.txt
│   ├── app.py
│   └── templates/
│       └── index.html          # Hostname is rendered from HOSTNAME_DISPLAY
└── polkit/
    └── 10-wifi-config.rules    # Template — username substituted by installer
```
