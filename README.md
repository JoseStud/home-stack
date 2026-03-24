# home-stack

Docker Compose stacks for home services, designed for a Raspberry Pi 5 host.

## Stacks

- `media/`: Prometheus Node Exporter, Plex, Dispatcharr, qBittorrent, Sonarr, Radarr, Prowlarr, Bazarr, FlareSolverr, Notifiarr
- `home-assistant/`: Home Assistant suite (Home Assistant, Mosquitto, Zigbee2MQTT, Z-Wave JS UI, Node-RED, ESPHome, Matter Server)
- `management/`: Host monitoring + Wake-on-LAN tools (Node Exporter, UpSnap)

## Usage

For any stack directory:

1. Copy the example env file: `cp .env.example .env`
2. Adjust values in `.env` for your system paths, timezone, and device mappings
3. Start the stack: `docker compose up -d`

## Management Stack

This stack is in `management/` and includes:

- `node-exporter`: Exposes host metrics for Prometheus scraping
- `upsnap`: Web UI for Wake-on-LAN device control

### Setup

1. `cd management`
2. `cp .env.example .env`
3. Create UpSnap data directory: `mkdir -p data`
4. Start services: `docker compose up -d`

### Service Access

- Node Exporter metrics: `http://<host-ip>:9100/metrics` (or your `NODE_EXPORTER_PORT`)
- UpSnap UI: `http://<host-ip>:8090` (default UpSnap UI port on host network mode)

### Notes

- `upsnap` uses host networking so it can send WOL magic packets to your LAN.
- `upsnap` requires `NET_RAW` capability for ping-based status checks.
