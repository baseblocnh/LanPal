# LanPal

[![Version](https://img.shields.io/badge/version-0.5_beta-blue.svg)](https://github.com/baseblocnh/LanPal/releases)
[![License: PolyForm Noncommercial 1.0.0](https://img.shields.io/badge/License-PolyForm%20Noncommercial%201.0.0-blue.svg)](LICENSE)

A self-hosted LAN tool. Open a browser on any device — desktop, phone, tablet, Shield TV, Echo Show — and measure real download speed, upload speed, latency, and jitter to servers on your local network, plus wake sleeping devices with a single click. No app installs, no extensions, no internet required after deployment.

## Features

- **Speed test** — three-phase test (latency, download 10 s, upload 10 s) entirely in the browser
- **History** — line chart + filterable results table, per device and server
- **Wake on LAN** — define targets, send magic packets, see live online/offline status; cross-VLAN support via companion relay or directed broadcast
- **Prometheus metrics** — `/metrics` endpoint with per-server and per-client labels
- **Grafana Alloy log forwarding** — push structured test results to Loki after each test
- **VLAN-friendly** — speed tests and server verification run in the browser; the main app never needs to reach companions directly

## How it works

```
Browser (any device)
  │
  ├─► Main App  (Docker, port 8080)
  │     Serves the web UI, stores results, relays WOL magic packets
  │
  └─► Companion (Docker or binary on each server you test to)
        Handles speed test traffic directly from the browser
        Also relays WOL magic packets for cross-VLAN wake
```

The speed test runs entirely between the browser and the companion — the main app is only involved for the UI and saving results.

## Prerequisites

- Docker on the server running the main app
- Portainer (optional, recommended for managing stacks)
- Docker or a standalone binary on each server you want to test to

## Deployment

### 1. Main App

Deploy once on any server. The web UI will be available at `http://<server-ip>:8080`.

**Via Portainer** — paste as a new stack:

```yaml
services:
  lanpal:
    image: ghcr.io/baseblocnh/lanpal:latest
    container_name: lanpal
    ports:
      - "8080:8080"
    volumes:
      - lanpal-data:/data
    environment:
      - DB_PATH=/data/lanpal.db
    restart: unless-stopped

volumes:
  lanpal-data:
```

**Via Docker CLI:**

```bash
docker run -d \
  --name lanpal \
  --restart unless-stopped \
  -p 8080:8080 \
  -v lanpal-data:/data \
  ghcr.io/baseblocnh/lanpal:latest
```

### 2. Companion

Deploy on **each server you want to test to**.

**Via Portainer:**

```yaml
services:
  lanpal-companion:
    image: ghcr.io/baseblocnh/lanpal-companion:latest
    container_name: lanpal-companion
    network_mode: host
    restart: unless-stopped
```

> `network_mode: host` is required for Wake-on-LAN relay. If you only need speed testing (no WOL relay), you can use `ports: ["5199:5199"]` instead.

**Via Docker CLI:**

```bash
docker run -d \
  --name lanpal-companion \
  --restart unless-stopped \
  --network host \
  ghcr.io/baseblocnh/lanpal-companion:latest
```

**As a standalone binary:**

Download the binary for your platform from the **Servers** page in the web UI:

| Platform | Command |
|---|---|
| Windows | Double-click `companion-windows-amd64.exe` — or run from PowerShell |
| Linux | `chmod +x companion-linux-amd64 && ./companion-linux-amd64` |
| Linux ARM64 (Pi) | `chmod +x companion-linux-arm64 && ./companion-linux-arm64` |
| macOS (Intel) | `chmod +x companion-darwin-amd64 && ./companion-darwin-amd64` |
| macOS (Apple Silicon) | `chmod +x companion-darwin-arm64 && ./companion-darwin-arm64` |

Change the port with `--port ./companion-linux-amd64 --port 9000`.

**Running as a Windows service (run as Administrator):**

```powershell
.\companion-windows-amd64.exe --install-service   # install + auto-start on boot
.\companion-windows-amd64.exe --remove-service    # uninstall
```

## Adding a Server

1. Open the web UI at `http://<main-app-ip>:8080`
2. Go to **Servers → Add Server**
3. Enter a name, the IP/hostname of the machine running the companion, and the port (default `5199`)
4. Click **Save & Verify** — your browser verifies the companion directly (works across VLANs)

## Running a Speed Test

1. Go to the **Speed Test** tab
2. Select a server from the dropdown
3. Click **Start Test** — runs latency ping, download (10 s), upload (10 s)
4. Results are saved automatically and visible in the **History** tab

## History

Each browser/device gets a unique ID automatically. Name your device by clicking the device pill in the top-right corner (e.g. "Shield TV", "iPhone 15", "Work Laptop"). Filter history by server or device using the dropdowns.

**Firewall requirements:**

| Connection | Port | Required for |
|---|---|---|
| Browser → Main App | 8080 | UI, saving results, reading history |
| Browser → Companion | 5199 | Speed test traffic |
| Main App → Companion | 5199 | Server verification, WOL relay |

## Wake on LAN

The **Wake** tab lets you define devices and wake them remotely. Two cross-VLAN delivery methods are supported:

**Option A — Companion relay (recommended)**

Select a companion on the **same VLAN as the target** from the "Relay via Companion" dropdown. No router configuration required.

```
Main App ──HTTP──► Companion (target VLAN) ──UDP broadcast──► Target device
```

**Option B — IP subnet directed broadcast**

Set the **Broadcast Address** to the directed broadcast of the target subnet (e.g. `192.168.20.255` for `192.168.20.0/24`). Requires `ip directed-broadcast` enabled on the router's VLAN interface.

```
Main App ──UDP──► Router ──L2 broadcast──► Target device
```

Status dots show online/offline state, polled every 30 seconds via TCP.

## Settings

**Prometheus Metrics** — enable the `/metrics` endpoint for scraping:

```yaml
scrape_configs:
  - job_name: lanpal
    static_configs:
      - targets: ['<main-app-ip>:8080']
```

**Grafana Alloy Log Forwarding** — push test results to Loki after each test. Requires a `loki.source.api` component in your Alloy config listening on the configured port.

**Logging** — set log verbosity (DEBUG / INFO / WARNING / ERROR) and view recent log entries directly in the Settings page.

## Multi-Architecture

Both images are built for `linux/amd64` and `linux/arm64` — runs natively on x86 servers, Raspberry Pi (3/4/5), and any arm64 NAS or SBC.

## Disclaimer

This software is provided **as-is**, without warranty of any kind. Speed test results are approximations based on HTTP throughput. Intended for use on private networks you own or have explicit permission to test.

## License

Copyright © 2025 baseblocnh. Licensed under the **PolyForm Noncommercial License 1.0.0** — see [LICENSE](LICENSE).

- ✅ Personal use, home labs, self-hosting
- ✅ Non-commercial organisations (education, research, charity, government)
- ❌ Commercial use of any kind
- ❌ Selling, licensing, or offering as a service

For commercial licensing enquiries, open an issue on GitHub.
