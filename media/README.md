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
3. Fill the environment variables in the Coolify UI — never in the repo: VPN credentials, `COOLIFY_NETWORK`, and the `SERVICE_URL_*` values (e.g. `SERVICE_URL_SONARR_8989=http://sonarr.justmammoth.us:8989`).
4. Deploy.

### If the deploy fails with `mutually exclusive network_mode and networks`

Coolify sometimes injects a `networks:` key into every service, which conflicts with `network_mode: service:gluetun` on prowlarr/transmission. Fix:

1. Switch the resource to **Raw Compose deployment mode** (full control of the file).
2. Confirm the Traefik proxy network name on the host: `docker network ls`. If it isn't `coolify`, set `COOLIFY_NETWORK` accordingly.
3. Redeploy.

### Routing notes

- Routing is managed by Coolify, not by Traefik labels in this file. sonarr/radarr/bazarr/jellyfin expose themselves via Coolify's `SERVICE_URL_<SERVICE>_<PORT>` magic env vars declared in their `environment:`; Coolify fills their values and generates the proxy routes.
- prowlarr/transmission share gluetun's network namespace and have no container IP of their own, so `SERVICE_URL_*` vars **cannot** route them. gluetun declares `expose: [9696, 9091]` as the routing target; their domains must be added **manually** in the Coolify UI (**Domains for gluetun**), then redeploy to regenerate the routes.
- In Coolify, fill the `SERVICE_URL_*` values (e.g. `http://sonarr.justmammoth.us:8989`) in the app's Environment Variables, then redeploy.
- `FIREWALL_OUTBOUND_SUBNETS` (default `172.18.0.0/16`) lets the VPN-backed containers reach Sonarr/Radarr on Coolify's network; narrow it if needed.
- gluetun publishes **no host ports** — the tunnel + Coolify proxy is the only exposure. PIA port forwarding is not enabled, so no inbound peer connections (outbound-only torrenting).

## Gotchas

- No auth on Sonarr/Radarr/Bazarr/Prowlarr/Transmission — anyone with network access can use them. Temporary by design; add auth before exposing publicly beyond the existing tunnel.
- Jellyfin mounts `/srv/media` read-only; add libraries in the Jellyfin UI pointing at `/media/movies`, `/media/tv`, `/media/music`.
- If gluetun restarts or the VPN drops, prowlarr/transmission lose network until it reconnects.
