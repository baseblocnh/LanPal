# LanPal

[![License: PolyForm Noncommercial 1.0.0](https://img.shields.io/badge/License-PolyForm%20Noncommercial%201.0.0-blue.svg)](LICENSE)

A self-hosted LAN tool. Open a browser on any device — desktop, phone, tablet, Shield TV, Echo Show — and measure real download speed, upload speed, latency, and jitter to servers on your local network, plus wake sleeping devices with a single click. No app installs, no extensions, no internet required after deployment.

## Features

- **Speed test** — three-phase test (latency, download 10 s, upload 10 s) entirely in the browser
- **History** — Chart.js line chart + filterable results table, per device and server
- **Wake on LAN** — define a list of targets, send magic packets, and see live online/offline status icons
- **Prometheus metrics** — `/metrics` endpoint with per-server and per-client labels
- **Grafana Alloy log forwarding** — push structured test results to Loki after each test
- **VLAN-friendly** — server verification runs in the browser; the main app never needs to reach companions directly

## How it works

```
Browser (any device)
  │
  ├─► Main App  (Docker on your server, port 8080)
  │     Serves the web UI, stores results, sends WOL magic packets
  │
  └─► Companion (Docker or binary on each server you test to)
        Handles download / upload / ping traffic directly from the browser
```

The speed test runs entirely between the browser and the companion — the main app is only involved for the UI and saving results. This means the measurement reflects the actual connection speed of whichever device is running the browser.

## Prerequisites

- Docker on the server running the main app
- Portainer (optional, but recommended for managing stacks)
- Docker or a standalone binary on each server you want to test to

> Container images are built automatically via GitHub Actions and published to `ghcr.io`. You do not need to build anything locally.

## Deployment

### 1. Main App

Deploy once on any server in your rack. The web UI will be available at `http://<server-ip>:8080`.

**Via Portainer** — paste `docker-compose.yml` as a new stack:

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

Deploy on **each server you want to test to** — including the same server as the main app if you want to test to it.

**Via Portainer** — paste `companion/docker-compose.yml` as a new stack:

```yaml
services:
  lanpal-companion:
    image: ghcr.io/baseblocnh/lanpal-companion:latest
    container_name: lanpal-companion
    ports:
      - "5199:5199"
    restart: unless-stopped
```

**Via Docker CLI:**

```bash
docker run -d \
  --name lanpal-companion \
  --restart unless-stopped \
  -p 5199:5199 \
  ghcr.io/baseblocnh/lanpal-companion:latest
```

**As a standalone binary (no Docker required):**

Download the binary for your platform from the **Get Companion** page in the web UI, then run it:

| Platform | Command |
|---|---|
| Windows | Double-click `companion-windows-amd64.exe` — or run from PowerShell |
| Linux | `chmod +x companion-linux-amd64 && ./companion-linux-amd64` |
| Linux ARM64 (Pi) | `chmod +x companion-linux-arm64 && ./companion-linux-arm64` |
| macOS (Intel) | `chmod +x companion-darwin-amd64 && ./companion-darwin-amd64` |
| macOS (Apple Silicon) | `chmod +x companion-darwin-arm64 && ./companion-darwin-arm64` |

Change the port with `--port`:

```bash
./companion-linux-amd64 --port 9000
```

## Adding a Server

1. Open the web UI at `http://<main-app-ip>:8080`
2. Go to **Servers → Add Server**
3. Enter a name, the IP/hostname of the machine running the companion, and the port (default `5199`)
4. Click **Save & Verify** — your browser verifies the companion directly (works across VLANs)

> **VLAN note:** Verification runs in the browser, not on the server, so it works even when the main app and the companion are on separate VLANs with one-way firewall rules.

## Running a Test

1. Go to the **Test** tab
2. Select a server chip
3. Click **Start Test**
4. The test runs three phases — latency ping, download (10 s), upload (10 s) — and displays results in real time
5. Results are automatically saved and visible in the **History** tab

## History

The **History** tab shows a Chart.js line chart of download speed over time per server, plus a full results table. Each browser/device gets a unique ID automatically; you can name your device by clicking the device pill in the top-right corner (e.g. "Shield TV", "iPhone 15", "Work Laptop").

Filter history by server or device using the dropdowns.

## Wake on LAN

The **Wake** tab lets you define a list of devices and wake them remotely:

1. Click **+ Add Target** and enter the device name, MAC address, IP, and broadcast address
2. Click **Wake** to send a magic packet from the main app server
3. Status dots show online (green) / offline (grey) state, checked every 30 seconds via TCP

> **VLAN note:** Magic packets and status checks are sent from the main app server. If your WOL targets are on a VLAN the main app cannot reach, ensure the appropriate firewall rules or directed broadcast routes are in place.

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

> **Note:** Metrics were renamed from `lantest_*` to `lanpal_*` in this release. Update any existing Prometheus rules or Grafana panels accordingly.

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

Use **Test Connection** in Settings to verify the endpoint is reachable before enabling.

## Multi-Architecture Support

Both Docker images are built for `linux/amd64` and `linux/arm64`. They run natively on:

- x86/x64 servers
- Raspberry Pi (3, 4, 5) running 64-bit OS
- Any arm64 NAS or SBC

## Building from Source

**Main app:**

```bash
cd app
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8080 --reload
```

**Companion:**

```bash
cd companion
go run .
# or build:
go build -o companion .
./companion --port 5199
```

## Project Structure

```
LanPal/
├── docker-compose.yml          # Main app — deploy via Portainer
├── app/
│   ├── Dockerfile              # Multi-arch Python image
│   ├── main.py                 # FastAPI backend
│   ├── database.py             # SQLite (servers, results, clients, settings, wol_targets)
│   ├── models.py               # Pydantic request models
│   ├── requirements.txt
│   └── static/
│       ├── index.html          # Single-page app shell
│       ├── style.css           # Dark/light themes, responsive layout
│       └── app.js              # All frontend logic
└── companion/
    ├── Dockerfile              # Multi-arch Go image (scratch-based, tiny)
    ├── docker-compose.yml      # Companion — deploy per server
    ├── go.mod
    └── main.go                 # HTTP test server (download / upload / ping)
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
