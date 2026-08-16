# home

Self-hosted home services, deployed with [Coolify](https://coolify.io).

## Services

### media/ — Media stack

A VPN-routed *arr stack and media server, with everything except VPN-bound apps exposed through Coolify's Traefik proxy.

| Service | Image | Port | Routing |
| --- | --- | --- | --- |
| gluetun | `qmcgaw/gluetun:v3` | 9091, 9696, 51413 | VPN gateway |
| prowlarr | `lscr.io/linuxserver/prowlarr:2.5.2` | 9696 | via gluetun (VPN) |
| transmission | `lscr.io/linuxserver/transmission:4.1.3` | 9091 | via gluetun (VPN) |
| sonarr | `lscr.io/linuxserver/sonarr:4.0.19` | 8989 | `https://sonarr.<DOMAIN>` |
| radarr | `lscr.io/linuxserver/radarr:6.3.0` | 7878 | `https://radarr.<DOMAIN>` |
| bazarr | `lscr.io/linuxserver/bazarr:1.6.0` | 6767 | `https://bazarr.<DOMAIN>` |
| jellyfin | `jellyfin/jellyfin:10.10` | 8096 | `https://jellyfin.<DOMAIN>` |

- `prowlarr` and `transmission` run on `network_mode: service:gluetun` so all of their traffic goes through the VPN.
- `sonarr`, `radarr`, `bazarr`, and `jellyfin` are labeled for Traefik routing on their own subdomains.
- All services expect media on the host under `/srv/media` (`tv`, `movies`, `music`, `downloads`, `subtitles`).

### home-assistant/ — Smart home stack

| Service | Image | Notes |
| --- | --- | --- |
| homeassistant | `ghcr.io/home-assistant/home-assistant:2026.4` | privileged, bind-mounts `configuration.yaml`, trusted proxy support for Coolify |
| mosquitto | `eclipse-mosquitto` | MQTT broker with persisted data |
| zigbee2mqtt | `ghcr.io/koenkk/zigbee2mqtt` | Sonoff Zigbee 3.0 USB dongle passthrough (`/dev/ttyUSB0`) |

## Deploying with Coolify

1. Add this repository to Coolify as a **Docker Compose** application (public repo works, or use a private-repo source).
2. For the media stack, copy the template and fill in the environment variables:

   ```bash
   cp media/.env.example media/.env
   ```

   Required values:

   | Variable | Purpose |
   | --- | --- |
   | `VPN_USER` / `VPN_PASS` | OpenVPN credentials (default provider: Private Internet Access) |
   | `DOMAIN` | Base domain for the Traefik routes (default `justmammoth.us`) |
   | `COOLIFY_NETWORK` | Coolify proxy network name (default `coolify`) |
   | `MEDIA_DIR` | Host path for media libraries (default `/srv/media`) |
   | `PUID` / `PGID` / `TZ` | LinuxServer.io standard user/group/timezone (defaults `1000` / `1000` / `Europe/Paris`) |
   | `SERVER_REGIONS` | Optional: restrict VPN to specific regions |
   | `FIREWALL_OUTBOUND_SUBNETS` | Outbound subnets allowed through the VPN firewall (defaults to the Coolify Docker subnet `172.18.0.0/16`) |

3. For the smart home stack, create the external network in Coolify first, then deploy `home-assistant/docker-compose.yaml`. Bind mounts for `configuration.yaml` and `mosquitto.conf` are provided inline in the compose file.
4. Ensure the host has the media directories available, e.g.:

   ```bash
   mkdir -p ${MEDIA_DIR:-/srv/media}/{tv,movies,music,downloads,subtitles}
   ```

## Notes

- The Traefik labels and the external network name (`coolify`) are set up to work with Coolify's proxy; adjust `COOLIFY_NETWORK` if you renamed it.
- Media libraries are mounted read-only into Jellyfin.
- Transmission's peer port `51413/tcp+udp` is published for incoming torrent connections.
