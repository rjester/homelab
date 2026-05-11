# Homelab

This repository is a Docker Compose-based homelab. Each service or service group lives in its own directory under `docker-compose/`, with Traefik providing ingress and Homepage acting as the primary dashboard.

The repo is oriented around host-managed state. Most stacks mount persistent data from absolute paths such as `/docker/...` and `/arr/data/...`, so it is intended to be deployed on a long-lived Linux host rather than run as a disposable local dev environment.

## Repository Layout

```text
docker-compose/
  adguard-exporter/
  adguard-sync/
  arrstack/
  barcodebuddy/
  dockpeek/
  fileflows/
  grafana/
  homepage/
    config/
  it-tools/
  traefik/
    config/
    data/
    secrets/
  velxio/
```

## Architecture

- `traefik/` is the edge proxy. It listens on ports `80` and `443`, uses the external Docker network `proxy`, and is configured for Cloudflare DNS challenge certificates.
- `homepage/` is the service dashboard. It includes config files for bookmarks, services, widgets, custom CSS, and custom JavaScript.
- Most user-facing stacks join the external `proxy` network and expose themselves through Traefik labels.
- `arrstack/` is the largest stack. It combines `gluetun`, `radarr`, `sonarr`, `prowlarr`, `bazarr`, `qbittorrent`, `flaresolverr`, and `seerr`.
- Persistent application data is generally stored outside the repo in host paths like `/docker/...` or `/arr/data/...`.

## Stacks

| Stack | Purpose | Notes |
| --- | --- | --- |
| `traefik` | Reverse proxy and TLS termination | Uses Cloudflare DNS challenge, mounts `config/`, `data/acme.json`, and a Docker socket proxy. |
| `homepage` | Landing page / service dashboard | Exposes port `8089` for direct access when Traefik is unavailable. |
| `arrstack` | Media automation and download pipeline | Routes several apps through `gluetun`; uses host media paths under `/arr/data`. |
| `grafana` | Dashboards and observability | Publicly routed through Traefik on the `proxy` network. |
| `fileflows` | Media processing / automation | Mounts movie and TV libraries directly from `/arr/data/media`. |
| `it-tools` | Utility toolbox | Exposes port `8088` locally and via Traefik. |
| `dockpeek` | Docker monitoring UI | Requires environment variables for auth and secret key. |
| `barcodebuddy` | Barcode ingestion for Grocy | Uses a persistent data mount under `/docker/barcodebuddy/data`. |
| `adguard-exporter` | Prometheus exporter for AdGuard | Publishes metrics on port `9618`; currently not routed through Traefik. |
| `adguard-sync` | Placeholder for AdGuard sync | `docker-compose.yml` is currently empty. |
| `velxio` | Additional web application | Minimal compose file with host data persisted to `/docker/velxio/data`. |

## Service Endpoints

| Service | Stack | Domain / Endpoint | Host Ports | Dependencies |
| --- | --- | --- | --- | --- |
| `traefik1` | `traefik` | Primary ingress for routed services | `80`, `443` | External `proxy` network, Cloudflare API token secret, Docker socket |
| `traefik-manager` | `traefik` | `traefik-manager.rjester.us` | None directly exposed | `traefik1`, `socket-proxy`, external `proxy` network |
| `socket-proxy` | `traefik` | Internal only | None | Docker socket |
| `dockerproxy` | `homepage` | Internal only | None | Docker socket, external `proxy` network |
| `homepage` | `homepage` | `home.rjester.us` | `8089:3000` | `dockerproxy`, external `proxy` network, local `config/` mount |
| `gluetun` | `arrstack` | VPN gateway for Arr services | `9696`, `7878`, `8989`, `6767`, `8080`, `6881/tcp`, `6881/udp`, `8191` | External `proxy` network, `/dev/net/tun`, VPN credentials |
| `radarr1` | `arrstack` | `radarr.rjester.us` | Via `gluetun` on `7878` | `gluetun` healthy, media and config mounts |
| `sonarr1` | `arrstack` | `sonarr.rjester.us` | Via `gluetun` on `8989` | `gluetun` healthy, media and config mounts |
| `prowlarr1` | `arrstack` | `prowlarr.rjester.us` | Via `gluetun` on `9696` | `gluetun` healthy, media and config mounts |
| `bazarr1` | `arrstack` | `bazarr.rjester.us` | Via `gluetun` on `6767` | `gluetun` healthy, media and config mounts |
| `qbittorrent1` | `arrstack` | `qbit.rjester.us` | Via `gluetun` on `8080`, torrenting on `6881/tcp+udp` | `gluetun` healthy, media and config mounts |
| `flaresolverr1` | `arrstack` | Internal only | Via `gluetun` on `8191` | `gluetun` healthy |
| `seerr1` | `arrstack` | `seerr.rjester.us` | `5055:5055` | External `proxy` network, config mount |
| `grafana` | `grafana` | `grafana.rjester.us` | None directly exposed | External `proxy` network, `/docker/grafana` volume |
| `fileflows` | `fileflows` | `fileflows.rjester.us` | None directly exposed | External `proxy` network, Docker socket, media mounts |
| `it-tools` | `it-tools` | `it-tools.rjester.us` | `8088:80` | External `proxy` network |
| `dockpeek` | `dockpeek` | `dockpeek.rjester.us` | `3420:8000` | External `proxy` network, Docker socket, env-provided auth |
| `barcodebuddy` | `barcodebuddy` | `barcodebuddy.rjester.us` | `8098:80` | Grocy API connectivity, persistent data volume |
| `henrywhitaker3` | `adguard-exporter` | Metrics endpoint only | `9618:9618` | Reachability to both AdGuard servers, env-provided credentials |
| `adguard-sync` | `adguard-sync` | Not configured | None | `docker-compose.yml` is empty |
| `velxio` | `velxio` | Not defined in compose labels | `3080:80`, `3081:8001` | `/docker/velxio/data` volume |

Notes:

- Arr applications except `seerr` share `gluetun` network mode, so their reachable host ports are published by `gluetun` rather than by each container directly.
- A blank `Host Ports` entry means the service is expected to be reached through Traefik only.
- Some external services appear in Homepage config but are not managed by compose in this repository.

## Homepage Config

Homepage configuration lives in `docker-compose/homepage/config/` and currently includes:

- `services.yaml` for categorized service links and widgets
- `settings.yaml` for theme, background, and layout
- `widgets.yaml` for global widgets like resources and search
- `bookmarks.yaml`, `docker.yaml`, `kubernetes.yaml`, and `proxmox.yaml` for additional integrations
- `custom.css` and `custom.js` for UI customization

The checked-in Homepage config shows this repo is intended to be the central dashboard for local infrastructure, media services, network tools, and home automation endpoints.

## Traefik Config

Traefik configuration lives in `docker-compose/traefik/config/`.

- `traefik.yml` defines entrypoints, the Docker provider, the file provider, and the Cloudflare certificate resolver.
- `middlewares.yml` defines shared middleware such as HTTPS redirection and security headers.
- Additional dynamic config files include `external-services.yml` and `transports.yml`.

The Traefik stack expects a secret file at `docker-compose/traefik/secrets/cf_api_token.txt` and a pre-existing Docker network named `proxy`.

## Deployment Model

This repo is organized as independent compose projects rather than one monolithic top-level compose file.

Bring up a single stack from its directory:

```bash
cd docker-compose/homepage
docker compose up -d
```

Or point Compose at a specific file from the repository root:

```bash
docker compose -f docker-compose/traefik/docker-compose.yml up -d
docker compose -f docker-compose/grafana/docker-compose.yml up -d
```

Useful operations:

```bash
docker compose -f docker-compose/homepage/docker-compose.yml pull
docker compose -f docker-compose/homepage/docker-compose.yml logs -f
docker compose -f docker-compose/homepage/docker-compose.yml ps
```

## Host Prerequisites

Before deploying, verify the host already has:

- Docker Engine with the Compose plugin
- An external Docker network named `proxy`
- Required host directories such as `/docker/...` and `/arr/data/...`
- Any required `.env` files or shell environment variables for stacks that reference `${...}` values
- Traefik secrets such as the Cloudflare API token file

Create the shared network if it does not already exist:

```bash
docker network create proxy
```

## CI/CD

The checked-in GitLab CI configuration targets Homepage deployment. The pipeline currently defines a `deploy_homepage` job and includes scaffolding for syncing Homepage config and updating the Homepage container over SSH, though the main deployment commands are commented out.

## Maintenance Notes

- This repository contains host-specific paths and domains such as `*.rjester.us` and local LAN addresses. Expect to adapt those values for another environment.
- Some checked-in compose and config files currently contain sensitive or environment-specific values. Review and externalize them into secrets or environment files before sharing or reusing this repo elsewhere.
- Because each stack is independent, upgrades and restarts are typically done per-directory rather than globally.

## Suggested Operating Order

For a fresh host, the practical order is:

1. Create the `proxy` network.
2. Provision host directories and secrets.
3. Start `traefik`.
4. Start `homepage`.
5. Bring up the remaining application stacks as needed.
