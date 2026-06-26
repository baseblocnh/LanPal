# LanPal

[![License: PolyForm Noncommercial 1.0.0](https://img.shields.io/badge/License-PolyForm%20Noncommercial%201.0.0-blue.svg)](LICENSE)

A self-hosted LAN tool. Open a browser on any device — desktop, phone, tablet, Shield TV, Echo Show — and measure real download speed, upload speed, latency, and jitter to servers on your local network, plus wake sleeping devices with a single click. No app installs, no extensions, no internet required after deployment.

## Features

- **Speed test** — three-phase test (latency, download 10 s, upload 10 s) entirely in the browser
- **Live waveform** — real-time time-series chart of download (blue) and upload (green) speed during the test, which stays visible afterward so you can see the shape of the connection
- **History** — Chart.js line chart + filterable results table, per device and server
- **Wake on LAN** — define targets, send magic packets, and see live online/offline status; cross-VLAN support via companion relay or unicast directed delivery
- **Prometheus metrics** — `/metrics` endpoint with per-server and per-client labels
- **Grafana Alloy log forwarding** — push structured test results to Loki after each test
- **VLAN-friendly** — server verification and speed tests run in the browser; the main app never needs to reach companions directly

## How it works

```
Browser (any device)
  │
  ├─► Main App  (Docker on your server, port 8080)
  │     Serves the web UI, stores results, relays WOL magic packets
  │
  └─► Companion (Docker or binary on each server you test to)
        Handles download / upload / ping traffic directly from the browser
        Also relays WOL magic packets on behalf of the main app (cross-VLAN)
```

The speed test runs entirely between the browser and the companion — the main app is only involved for the UI and saving results. This means the measurement reflects the actual connection speed of whichever device is running the browser.

## Prerequisites

- Docker on the server running the main app
- Portainer (optional, but recommended for managing stacks)
- Docker or a standalone binary on each server you want to test to

> Container images are built automatically via GitHub Actions and published to `ghcr.io`. You do not need to build anything locally. Packages are located here: https://github.com/baseblocnh?tab=packages

## Deployment

### 1. Main App

Deploy once on any server in your rack. The web UI will be available at `http://<server-ip>:8080`.

**Via Portainer** — paste as a new stack:

```yaml
services:
  lanpal:
    image: ghcr.io/baseblocnh/lanpal:latest
    container_name: lanpal
    network_mode: host
    volumes:
      - lanpal-data:/data
    environment:
      - DB_PATH=/data/lanpal.db
    restart: unless-stopped

volumes:
  lanpal-data:
```

> `network_mode: host` is required for Wake-on-LAN direct mode to reach your LAN. The app binds to port 8080 directly on the host.

**Via Docker CLI:**

```bash
docker run -d \
  --name lanpal \
  --restart unless-stopped \
  --network host \
  -v lanpal-data:/data \
  ghcr.io/baseblocnh/lanpal:latest
```

### 2. Companion

Deploy on **each server you want to test to** — including the same server as the main app if you want to test to it.

**Via Portainer** — paste as a new stack:

```yaml
services:
  lanpal-companion:
    image: ghcr.io/baseblocnh/lanpal-companion:latest
    container_name: lanpal-companion
    network_mode: host
    restart: unless-stopped
```

> `network_mode: host` is required for the companion to send WOL broadcast packets onto the physical LAN. The companion binds to port 5199 directly on the host.

**Via Docker CLI:**

```bash
docker run -d \
  --name lanpal-companion \
  --restart unless-stopped \
  --network host \
  ghcr.io/baseblocnh/lanpal-companion:latest
```

**As a standalone binary (no Docker required):**

Download the binary for your platform from the **Servers** page in the web UI:

| Platform | Download | Notes |
|---|---|---|
| Windows | `LanPalCompanion-Setup.msi` | Installs as a Windows service, auto-starts on boot |
| Linux x64 | `companion-linux-amd64` | x86/x64 servers, VMs, Intel/AMD NAS |
| Linux ARM64 | `companion-linux-arm64` | Raspberry Pi 3/4/5, arm64 NAS/SBCs |
| Linux ARMv7 | `companion-linux-armv7` | Synology DS220j/DS418 and other ARMv7 NAS, RPi 2 |
| Linux ARMv5 | `companion-linux-armv5` | Synology DS213 and older Marvell ARMv5 NAS (no Docker) |
| macOS Intel | `companion-darwin-amd64` | `chmod +x` then run |
| macOS Apple Silicon | `companion-darwin-arm64` | `chmod +x` then run |

> **Synology NAS:** older models (DS213 and similar) run a Marvell **ARMv5**
> CPU and cannot run Docker at all — use the `companion-linux-armv5` binary.
> ARMv7 models (DS220j, DS418, etc.) can run Docker or use `companion-linux-armv7`.
> Newer Intel/AMD models use `companion-linux-amd64` or the Docker image.
> If you see `exec format error`, you are running the wrong architecture's binary.

**Windows MSI**: run the installer — it installs LanPal Companion as a Windows service (auto-starts on boot), creates a desktop shortcut with the LanPal icon, and adds a Start Menu entry. Uninstall cleanly via *Add or Remove Programs*.

**Linux / macOS:**

```bash
chmod +x companion-linux-amd64
./companion-linux-amd64 --port 5199
```

Change the port with `--port`:

```bash
./companion-linux-amd64 --port 9000
```

## Adding a Server

1. Open the web UI at `http://<main-app-ip>:8080`
2. Go to **Servers → Add Server**
3. Enter a name, the IP/hostname of the machine running the companion, and the port (default `5199`)
4. Click **Save & Verify** — your browser verifies the companion directly (works across VLANs)

> **VLAN note:** Verification runs in the browser, not on the server, so it works even when the main app and companion are on separate VLANs.

## Running a Speed Test

1. Go to the **Speed Test** tab
2. Pick a server from the list — each row shows its live online/offline status so you can choose a reachable one before starting
3. Click **Start Test**
4. The test runs three phases — latency ping, download (10 s), upload (10 s) — and displays results in real time
5. A live waveform chart plots download (blue) and upload (green) speed as the test runs and remains on screen afterward for inspection
6. Results are automatically saved and visible in the **History** tab

## History

The **History** tab shows a line chart of download speed over time per server, plus a full results table. Each browser/device gets a unique ID automatically; name your device by clicking the device pill in the top-right corner (e.g. "Shield TV", "iPhone 15", "Work Laptop").

Filter history by server or device using the dropdowns.

**Requirements for history to work:**

| Connection | Required for |
|---|---|
| Browser → Main App (port 8080) | Loading the UI, saving results, reading history |
| Browser → Companion (port 5199) | Speed test traffic |
| Main App → Companion (port 5199) | WOL relay |

If your browser is on a different VLAN from the main app, ensure a firewall rule allows traffic from the browser VLAN to the main app on port 8080. Without this, test results cannot be saved and history will remain empty.

## Wake on LAN

The **Wake** tab lets you define a list of devices and wake them remotely.

### Adding a target

1. Click **+ Add Target**
2. Enter the device name, MAC address, IP address (for status checks), and WOL port (default `9`)
3. Optionally set the subnet broadcast address (e.g. `192.168.20.255`) and status check port (default `9`)
4. Choose a relay companion if the target is on a different VLAN
5. Click **Save**

### Cross-VLAN WOL

Magic packets are sent to multiple destinations simultaneously to maximise compatibility:

- Subnet broadcast (e.g. `192.168.20.255`) on ports 9 and 7
- Global broadcast `255.255.255.255` on ports 9 and 7
- **Unicast directly to the device IP** on ports 9 and 7

The unicast delivery is the key cross-VLAN mechanism — routers forward unicast UDP packets normally, and modern NICs accept unicast magic packets. No router configuration required.

**Option A — Companion relay (recommended for cross-VLAN)**

Select a companion running on the **same VLAN as the target** from the "Relay via Companion" dropdown. The main app forwards the wake request to that companion over HTTP; the companion sends all packets locally.

```
Main App ──HTTP──► Companion (target VLAN) ──UDP──► Target device
```

**Option B — Direct (main app sends packets)**

Leave the relay companion unset. The main app sends all six packets directly. With `network_mode: host` on the main app, the unicast packet to the device IP will cross VLAN boundaries through the router.

```
Main App ──UDP unicast──► Router ──► Target device
```

### Status indicators

Status dots on each target show online (green) / offline (grey) state, polled every 30 seconds.

Detection uses **ICMP ping** first — the main app sends a single echo request to the target IP. This is the most reliable LAN liveness signal because it does not depend on any TCP port being open.

> **Firewall:** allow ICMP echo from the LanPal host to your target devices (and, for cross-VLAN targets, on the relevant router/firewall interfaces). Windows devices also need *File and Printer Sharing (Echo Request – ICMPv4-In)* enabled to answer ping.

If ICMP fails (blocked or no reply), it falls back to a **TCP connection** on the configured **status check port** — a port that **accepts** or **actively refuses** (sends a reset) counts as online; a timeout counts as offline. Point this fallback port at a service the device keeps open:

| Target | Fallback check port |
|---|---|
| Linux / NAS | `22` (SSH) |
| Windows PC | `445` (SMB) |
| Web server | `80` or `443` |

> A closed/filtered port such as `9` on a firewalled host will time out and read as offline, so the TCP fallback only helps if it targets a listening service.

## Settings

### Prometheus Metrics

Enable the `/metrics` endpoint in **Settings → Prometheus Metrics**. When enabled, the endpoint is available at:

```
http://<main-app-ip>:8080/metrics
```

Metrics exposed:

| Metric | Type | Labels | Description |
|---|---|---|---|
| `lanpal_last_download_mbps` | Gauge | server_name, server_host, client_name | Last download speed |
| `lanpal_last_upload_mbps` | Gauge | server_name, server_host, client_name | Last upload speed |
| `lanpal_last_latency_ms` | Gauge | server_name, server_host, client_name | Last latency |
| `lanpal_last_jitter_ms` | Gauge | server_name, server_host, client_name | Last jitter |
| `lanpal_tests_total` | Gauge | server_name, server_host, client_name | Total tests per server+client |

Add to your Prometheus `scrape_configs`:

```yaml
scrape_configs:
  - job_name: lanpal
    static_configs:
      - targets: ['<main-app-ip>:8080']
```

### Grafana Alloy Log Forwarding

Enable in **Settings → Grafana Alloy Log Forwarding** and enter your Alloy host/IP and port (default `3100`). After each test, a structured log entry is pushed to:

```
http://<alloy-host>:<port>/loki/api/v1/push
```

Your Alloy config needs a `loki.source.api` component listening on that port:

```hcl
loki.source.api "lanpal" {
  http {
    listen_address = "0.0.0.0"
    listen_port    = 3100
  }
  forward_to = [loki.write.default.receiver]
}
```

## Multi-Architecture Support

The **main app** image is built for `linux/amd64` and `linux/arm64`.
The **companion** image is built for `linux/amd64`, `linux/arm64`, and `linux/arm/v7`,
so ARMv7 NAS devices that support Docker pull the correct image automatically.

They run natively on:

- x86/x64 servers
- Raspberry Pi (2 via arm/v7; 3/4/5 via arm64)
- arm64 and ARMv7 NAS or SBCs

For older ARMv5 hardware that cannot run Docker (e.g. Synology DS213), download
the standalone `companion-linux-armv5` binary from the **Servers** page instead.

## Project Structure

```
LanPal/
├── app/
│   ├── Dockerfile              # Multi-arch Python image
│   ├── main.py                 # FastAPI backend
│   ├── database.py             # SQLite (servers, results, clients, settings, wol_targets)
│   ├── models.py               # Pydantic request models
│   ├── requirements.txt
│   └── static/
│       ├── index.html          # Single-page app shell
│       ├── style.css           # Dark/light themes, responsive layout
│       ├── app.js              # All frontend logic
│       └── favicon.svg         # App icon
└── companion/
    ├── Dockerfile              # Multi-arch Go image (Alpine-based)
    ├── go.mod
    ├── main.go                 # HTTP test server (download / upload / ping / wol)
    ├── broadcast_unix.go       # SO_BROADCAST socket option (Linux/macOS)
    ├── broadcast_windows.go    # SO_BROADCAST socket option (Windows)
    ├── service_windows.go      # Windows service (SCM) + setup wizard
    ├── service_other.go        # No-op service stubs (Linux/macOS)
    ├── icon_windows.go         # Embedded icon generation (Windows)
    └── installer/
        ├── companion.wxs       # WiX v4 MSI source
        ├── license.rtf         # Acceptable Use Policy shown in the installer
        └── gen-icon/           # Go tool — generates .ico, banner.bmp, dialog.bmp
```

## Disclaimer

This software is provided **as-is**, without warranty of any kind, express or implied. The author makes no guarantees regarding accuracy, reliability, fitness for a particular purpose, or uninterrupted operation.

- Speed test results are approximations based on HTTP throughput and may not reflect the maximum theoretical capacity of your network hardware.
- The author is not responsible for any network disruption, data loss, or other issues arising from the use of this software.
- This tool is intended for use on private networks you own or have explicit permission to test. Do not deploy the companion on networks or devices without the knowledge and consent of the network administrator.
- Running high-throughput speed tests may temporarily impact other devices sharing the same network segment.

## License

Copyright © 2025 baseblocnh

This project is licensed under the **PolyForm Noncommercial License 1.0.0**. See [LICENSE](LICENSE) for the full text.

**In plain terms:**
- ✅ Personal use, home labs, self-hosting — permitted
- ✅ Non-commercial organisations (education, research, charity, government) — permitted
- ✅ Modify and build upon the code for personal/non-commercial purposes — permitted
- ❌ Commercial use of any kind — not permitted
- ❌ Selling, licensing, or offering this software as a service — not permitted

For commercial licensing enquiries, open an issue on GitHub.
