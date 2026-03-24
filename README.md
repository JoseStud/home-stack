# home-stack

Docker Compose stacks for home services, designed for a Raspberry Pi 5 host.

## Stacks

- `media/`: Prometheus Node Exporter, Plex, Dispatcharr, qBittorrent, Sonarr, Radarr, Prowlarr, Bazarr, FlareSolverr, Notifiarr
- `home-assistant/`: Home Assistant suite (Home Assistant, Mosquitto, Zigbee2MQTT, Z-Wave JS UI, Node-RED, ESPHome, Matter Server)

## Usage

For any stack directory:

1. Copy the example env file: `cp .env.example .env`
2. Adjust values in `.env` for your system paths, timezone, and device mappings
3. Start the stack: `docker compose up -d`
