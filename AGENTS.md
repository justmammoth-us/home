# AGENTS.md

## What this repo is

Docker Compose stacks for self-hosted home services, deployed via Coolify:

- `media/` — gluetun-routed *arr stack (prowlarr, transmission, sonarr, radarr, bazarr, jellyfin)
- `home-assistant/` — homeassistant, mosquitto, zigbee2mqtt

No build step, no tests, no lint. Validation is `docker compose config` against the compose files (with required env vars supplied).

## Critical Coolify constraint

**Never use `${VAR:-default}` syntax inside `volumes:` mount paths.** Coolify's compose preprocessor splits volume specs on `:` and mangles `:-`, producing errors like `invalid spec: /srv/media::`. Volume host paths must use plain `${VAR}` (e.g. `${MEDIA_DIR}/downloads:/downloads`). Defaults are fine in `environment:` and anywhere else.

`DOMAIN` and `MEDIA_DIR` are intentionally **required** (no defaults) — they're set in the Coolify UI, never hardcoded.

## Image conventions

- Registries: `ghcr.io` and `lscr.io` only. Do **not** reintroduce Docker Hub images (`qmcgaw/gluetun`, `jellyfin/jellyfin`) — the repo moved off Docker Hub to avoid anonymous pull rate limits during deployments.
- LinuxServer images use `lscr.io/linuxserver/<app>:<version>` (e.g. `jellyfin:10.10.7`). The LinuxServer jellyfin build is used instead of the official image; it takes `PUID`/`PGID`/`TZ`.
- **All image tags must be pinned to exact versions** — Renovate (see `renovate.json`) manages updates and cannot manage floating tags like `:v3` or `:latest`. Prefer the `X.X.X` or `vX.Y.Z` tag; verify the tag exists on the registry before using it.

## Media stack routing model

- `prowlarr` and `transmission` run with `network_mode: 'service:gluetun'` — all their traffic exits via the VPN. They `depends_on: gluetun (condition: service_healthy)`.
- Their Traefik labels live on the **gluetun** service, since they share its network namespace. Direct services (sonarr/radarr/bazarr/jellyfin) carry their own Traefik labels.
- Traefik label pattern per service: `Host(\`<svc>.${DOMAIN}\`)`, entrypoint `https`, `tls=true`, service name = compose service, port = app port (e.g. sonarr 8989, jellyfin 8096).
- All services share one external network `name: ${COOLIFY_NETWORK:-coolify}`.
- Media paths: `${MEDIA_DIR}/{movies,tv,music,downloads,subtitles}`. Jellyfin mounts `movies`/`tv`/`music` **read-only** under `/media/*`.
- Transmission peer port `51413/tcp+udp` is published on gluetun.

## Environment

- `media/.env.example` is the template; real values live in the Coolify UI, never in the repo.
- Standard vars: `PUID`/`PGID` (1000), `TZ` (Europe/Paris), `MEDIA_DIR` (/srv/media), `DOMAIN` (justmammoth.us), `COOLIFY_NETWORK` (coolify), `FIREWALL_OUTBOUND_SUBNETS` (172.18.0.0/16), `VPN_*`/`SERVER_REGIONS` for gluetun.

## Renovate

`renovate.json` at the repo root: `config:recommended`, weekday schedule, LinuxServer image updates grouped into one PR, no auto-merge (Coolify redeploys on push to main). Keep image pins exact so Renovate sees them.

## Conventions

- Compose files are named `docker-compose.yaml`.
- YAML: 2-space indent, quoted ports, flow-style arrays for healthcheck `test:`.
- Every service: `restart: unless-stopped`; curl/wget-based healthchecks on direct services.
- Commits: Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`), no emoji.
