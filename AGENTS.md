# AGENTS.md

## What this repo is

Docker Compose stacks for self-hosted home services, deployed via Coolify:

- `media/` — gluetun-routed *arr stack (prowlarr, transmission, sonarr, radarr, bazarr, jellyfin)
- `home-assistant/` — homeassistant, mosquitto, zigbee2mqtt

No build step, no tests, no lint. Validation is `docker compose config` against the compose files (with required env vars supplied).

## Critical Coolify constraint

**Never use `${VAR:-default}` syntax inside `volumes:` mount paths.** Coolify's compose preprocessor splits volume specs on `:` and mangles `:-`, producing errors like `invalid spec: /srv/media::`. Volume host paths must use plain `${VAR}` (e.g. `${MEDIA_DIR}/downloads:/downloads`). Defaults are fine in `environment:` and anywhere else.

`MEDIA_DIR` is intentionally **required** (no default) — it's set in the Coolify UI, never hardcoded.

## Image conventions

- Registries: `ghcr.io` and `lscr.io` only. Do **not** reintroduce Docker Hub images (`qmcgaw/gluetun`, `jellyfin/jellyfin`) — the repo moved off Docker Hub to avoid anonymous pull rate limits during deployments.
- LinuxServer images use `lscr.io/linuxserver/<app>:<version>` (e.g. `jellyfin:10.10.7`). The LinuxServer jellyfin build is used instead of the official image; it takes `PUID`/`PGID`/`TZ`.
- **All image tags must be pinned to exact versions** — Renovate (see `renovate.json`) manages updates and cannot manage floating tags like `:v3` or `:latest`. Prefer the `X.X.X` or `vX.Y.Z` tag; verify the tag exists on the registry before using it.

## Media stack routing model

- `prowlarr` and `transmission` run with `network_mode: 'service:gluetun'` — all their traffic exits via the VPN. They `depends_on: gluetun (condition: service_healthy)`.
- Routing is Coolify-managed, **not** file Traefik labels (Coolify ignores those for routing and its own generated labels carry `traefik.docker.network`). sonarr/radarr/bazarr/jellyfin declare `SERVICE_URL_<SERVICE>_<PORT>` (e.g. `SERVICE_URL_SONARR_8989`) in their `environment:`; values live in the Coolify UI.
- prowlarr/transmission share gluetun's network namespace and have no container IP, so `SERVICE_URL_*` vars cannot route them. Set **Domains for gluetun** in the Coolify UI: `http://prowlarr.justmammoth.us:9696,http://transmission.justmammoth.us:9091`.
- gluetun publishes **no host ports**. It only masks the IP of prowlarr/transmission; PIA port forwarding is not enabled, so no inbound peer connections (outbound-only torrenting, fine).
- All services share one external network `name: ${COOLIFY_NETWORK:-coolify}`.
- Media paths: `${MEDIA_DIR}/{movies,tv,music,downloads,subtitles}`. Jellyfin mounts `movies`/`tv`/`music` **read-only** under `/media/*`.

## Environment

- `media/.env.example` is the template; real values live in the Coolify UI, never in the repo.
- Standard vars: `PUID`/`PGID` (1000), `TZ` (Europe/Paris), `MEDIA_DIR` (/srv/media), `COOLIFY_NETWORK` (coolify), `FIREWALL_OUTBOUND_SUBNETS` (172.18.0.0/16), the `SERVICE_URL_<SERVICE>_<PORT>` values, and `VPN_*`/`SERVER_REGIONS` for gluetun. `MEDIA_DIR` is required (no default) — set in the Coolify UI.

## Renovate

`renovate.json` at the repo root: `config:recommended`, weekday schedule, LinuxServer image updates grouped into one PR, no auto-merge (Coolify redeploys on push to main). Keep image pins exact so Renovate sees them.

## Conventions

- Compose files are named `docker-compose.yaml`.
- YAML: 2-space indent, quoted ports, flow-style arrays for healthcheck `test:`.
- Every service: `restart: unless-stopped`; curl/wget-based healthchecks on direct services.
- Commits: Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`), no emoji.
