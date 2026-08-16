# Media stack — temporary Docker Compose

Temporary media stack deployable via Coolify or plain Docker Compose. No JuiceFS, no Redis, no Traefik/cloudflared (the existing `*.justmammoth.us` reverse proxy handles exposure). No authentication on the *arr apps, only Jellyfin keeps its own login.

## Services

| Service | Port | Domain | Network |
|---------|------|--------|---------|
| gluetun | — | — | VPN gateway |
| prowlarr | 9696 | `prowlarr.justmammoth.us` | behind gluetun |
| transmission | 9091 | `transmission.justmammoth.us` | behind gluetun |
| sonarr | 8989 | `sonarr.justmammoth.us` | direct |
| radarr | 7878 | `radarr.justmammoth.us` | direct |
| bazarr | 6767 | `bazarr.justmammoth.us` | direct |
| jellyfin | 8096 | `jellyfin.justmammoth.us` | direct |

Media lives on disk under `/srv/media/{movies,tv,music,downloads,subtitles}`.

## Host preparation

```bash
sudo mkdir -p /srv/media/{movies,tv,music,downloads,subtitles}
sudo chown -R 1000:1000 /srv/media
```

(Adjust `1000:1000` to match `PUID`/`PGID` in your env.)

## Local run (no Coolify)

```bash
cp .env.example .env
# fill in VPN_SERVICE_PROVIDER, VPN_USER, VPN_PASS (PIA credentials)
docker compose up -d
```

`COOLIFY_NETWORK` is only used under Coolify; for a plain local run it must point to an existing Docker network or you can ignore the default (`coolify`).

## Coolify deployment

1. Create a project → **Docker Compose** resource.
2. Paste the contents of `docker-compose.yml` (or point the resource at this repo).
3. Fill the environment variables (VPN credentials, `DOMAIN`, `COOLIFY_NETWORK`) in the Coolify UI — never in the repo.
4. Deploy.

### If the deploy fails with `mutually exclusive network_mode and networks`

Coolify sometimes injects a `networks:` key into every service, which conflicts with `network_mode: service:gluetun` on prowlarr/transmission. Fix:

1. Switch the resource to **Raw Compose deployment mode** (full control of the file).
2. Confirm the Traefik proxy network name on the host: `docker network ls`. If it isn't `coolify`, set `COOLIFY_NETWORK` accordingly.
3. Redeploy.

### Routing notes

- Do **not** set domains in the Coolify UI for these services — routing is defined by the Traefik labels inside `docker-compose.yml` (single source of truth).
- prowlarr/transmission share gluetun's network namespace, so their Traefik labels live on the **gluetun** container. Traefik reaches them over the Docker network.
- `FIREWALL_OUTBOUND_SUBNETS` (default `172.18.0.0/16`) lets the VPN-backed containers reach Sonarr/Radarr on Coolify's network; narrow it if needed.
- Transmission peer port `51413/tcp+udp` is published for incoming torrent connections.

## Gotchas

- No auth on Sonarr/Radarr/Bazarr/Prowlarr/Transmission — anyone with network access can use them. Temporary by design; add auth before exposing publicly beyond the existing tunnel.
- Jellyfin mounts `/srv/media` read-only; add libraries in the Jellyfin UI pointing at `/media/movies`, `/media/tv`, `/media/music`.
- If gluetun restarts or the VPN drops, prowlarr/transmission lose network until it reconnects.
