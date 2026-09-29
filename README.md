# home-stack

Docker Compose stacks for home services, designed for a Raspberry Pi 5 host.

## Stacks

- `media/`: Prometheus Node Exporter, Plex, Dispatcharr, qBittorrent, Sonarr, Radarr, Prowlarr, Bazarr, FlareSolverr, Notifiarr
- `home-assistant/`: Home Assistant suite (Home Assistant, Mosquitto, Zigbee2MQTT, Z-Wave JS UI, Node-RED, ESPHome, Matter Server)
- `management/`: Host monitoring + Wake-on-LAN tools (Node Exporter, UpSnap)
- `n8n/`: Workflow automation (n8n)

The media library itself lives on a phone, not on the Pi. See
[docs/phone-media-server.md](docs/phone-media-server.md).

## Usage

For any stack directory:

1. Copy the example env file: `cp .env.example .env`
2. Adjust values in `.env` for your system paths, timezone, and device mappings
3. Start the stack: `docker compose up -d`

## Media Stack

This stack is in `media/`. `MEDIA_DIR` (default `/mnt/media`) holds `movies/`
and `tv/`, and `DOWNLOADS_DIR` holds qBittorrent's downloads.

### Setup (media)

1. `cd media`
2. `cp .env.example .env`
3. Mount the media folders at `MEDIA_DIR/movies` and `MEDIA_DIR/tv`. In the
   author's setup these are sshfs mounts of a phone's storage, which also runs
   a standby Plex server: see [docs/phone-media-server.md](docs/phone-media-server.md).
4. Start services: `docker compose up -d`

### Service Access (media)

Plex `:32400`, Dispatcharr `:9191`, qBittorrent `:8080`, Sonarr `:8989`,
Radarr `:7878`, Prowlarr `:9696`, Bazarr `:6767`, FlareSolverr `:8191`.

### Notes

- Sonarr, Radarr and Bazarr break if `MEDIA_DIR` is unreachable at start. If
  the mount source moves, remount first and then start them.

## Management Stack

This stack is in `management/` and includes:

- `node-exporter`: Exposes host metrics for Prometheus scraping
- `upsnap`: Web UI for Wake-on-LAN device control

### Setup (management)

1. `cd management`
2. `cp .env.example .env`
3. Create UpSnap data directory: `mkdir -p data`
4. Start services: `docker compose up -d`

### Service Access (management)

- Node Exporter metrics: `http://<host-ip>:9100/metrics` (or your `NODE_EXPORTER_PORT`)
- UpSnap UI: `http://<host-ip>:8090` (default UpSnap UI port on host network mode)

### Notes

- `upsnap` uses host networking so it can send WOL magic packets to your LAN.
- `upsnap` requires `NET_RAW` capability for ping-based status checks.

## n8n Stack

This stack is in `n8n/` and includes:

- `n8n`: Workflow automation and integrations

### Setup

1. `cd n8n`
2. `cp .env.example .env`
3. Start services: `docker compose up -d`

### Notes

- n8n data is stored in the Docker named volume `n8n_data` to avoid host path permission issues.

### Service Access

- n8n UI: `http://<host-ip>:5678` (or your configured `N8N_PORT`)
