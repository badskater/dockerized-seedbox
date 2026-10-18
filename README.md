# dockerized-seedbox

Self-hosted Docker Compose stacks for a homelab, designed to run behind
[Traefik](https://traefik.io/) with [Authelia](https://www.authelia.com/) SSO
in front of every web UI.

Each directory is an independent stack: `cd` into it, copy `.env.example` to
`.env`, fill it in, and run `docker compose up -d`.

```
.
├── quickstart/          # Minimal Dockhand on port 3000 — no Traefik needed
├── traefik/             # Reverse proxy + Cloudflare DNS-01 Let's Encrypt certs
├── authelia/            # SSO portal (forwardAuth middleware for all stacks)
├── dockhand/            # Docker web UI (Traefik-routed, postgres-backed)
├── qbittorrent/         # Torrent client + WebUI (behind Authelia)
├── rutorrent-user1/    # rTorrent + ruTorrent (main instance)
├── rutorrent-user2/    # rTorrent + ruTorrent (2nd instance, offset ports)
├── sftpgo/              # SFTP / WebDAV / FTP server with web admin
└── filezilla/           # FileZilla desktop GUI in the browser (Selkies)
```

---

## Quickstart — Dockhand on port 3000

The fastest path: a Docker web UI backed by Postgres, published directly on
the host at **http://localhost:3000**. No reverse proxy, no TLS, no DNS.
Dockhand app data is bind-mounted from `./data` next to the compose file —
change that folder to wherever you want it.

```bash
cd quickstart
docker compose up -d
```

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: dockhand
      POSTGRES_PASSWORD: changeme
      POSTGRES_DB: dockhand
    volumes:
      - postgres_data:/var/lib/postgresql/data

  dockhand:
    image: fnsys/dockhand:latest
    ports:
      - 3000:3000
    environment:
      DATABASE_URL: postgres://dockhand:changeme@postgres:5432/dockhand
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      # Change ./data to wherever you want Dockhand app data on the host
      - ./data:/app/data
    depends_on:
      - postgres
    restart: unless-stopped

volumes:
  postgres_data:
```

**Security note:** mounting `/var/run/docker.sock` grants root-equivalent
access to the host. This quickstart is meant for a trusted LAN or a quick
evaluation — for anything exposed to the internet, use the full setup below
with TLS and Authelia in front. Change `changeme` before deploying.

---

## Full setup — Traefik + Authelia + app stacks

### Architecture

```
internet ──► Traefik (443, Cloudflare DNS-01 certs)
                 │  docker network: proxy
                 ├──► Authelia portal        (id.example.com)
                 ├──► Dockhand UI            (dockhand.example.com)   ──► postgres (private net)
                 ├──► qBittorrent WebUI      (qbit.example.com)
                 ├──► ruTorrent ×2           (rutorrent-*.example.com)
                 ├──► SFTPGo web admin       (sftp.example.com)
                 └──► FileZilla GUI          (fz.example.com)
```

Every web app is reached **only** through Traefik (`expose`, never `ports`),
and every router can be locked behind the shared `authelia@docker` forwardAuth
middleware.

### Prerequisites

- Docker + Docker Compose v2 on a Linux host
- A domain in Cloudflare (DNS-01 challenges issue real public certs; they also
  work for LAN-only hostnames via DNS-only records)
- A scoped Cloudflare API token: **Zone → DNS → Edit** for your zone
- Ports 80/443 reachable from the internet (or use split-horizon DNS +
  DNS-01, which needs no inbound 80)

### 1. Shared network

All stacks attach to one external network:

```bash
docker network create proxy
```

### 2. Traefik (`traefik/`)

```bash
cd traefik
cp .env.example .env
# fill in: CF_DNS_API_TOKEN, ACME_EMAIL, DASHBOARD_HOST, WHOAMI_HOST
# ACME_DIR / DYNAMIC_DIR / CERTS_DIR default to ./certs / ./dynamic relative
# to the stack dir — point them wherever you keep Traefik state
docker compose up -d
```

- Entrypoints: `web` (:80, redirects to HTTPS), `websecure` (:443), `traefik` (:8080, internal)
- Cert resolver `letsencrypt` via Cloudflare DNS-01 — every stack below
  references it by name
- The dashboard + a whoami test app are routed behind Authelia (see step 3 —
  Authelia must be up before these routers resolve their middleware)

Verify: `curl -sI https://whoami.<your-domain>` should return a redirect to
the Authelia portal once step 3 is done.

### 3. Authelia (`authelia/`)

```bash
cd authelia
cp .env.example .env
# fill in: PORTAL_HOST (e.g. id.example.com), storage/JWT secrets per the
# comments, and create configuration.yml (users + access_control rules)
docker compose up -d
```

Gotchas baked into this stack:

- Compose-level vars are named `PORTAL_*`, **not** `AUTHELIA_*` — Authelia
  treats every `AUTHELIA_*` env var as a config-key override and it silently
  breaks `server.address`
- Healthcheck probes the API via wget (`authelia healthcheck` doesn't exist
  in v4.39)
- Define `access_control` rules for every hostname you plan to route
  (traefik dashboard, whoami, dockhand, qbittorrent, rutorrent, sftpgo,
  filezilla)

Authelia must be **up before** the other stacks' routers try to use
`authelia@docker`, otherwise Traefik logs "middleware not found".

### 4. App stacks

Same pattern for each: `cp .env.example .env`, edit, `docker compose up -d`.

| Stack | Hostname var(s) | Notes |
|---|---|---|
| `dockhand/` | `DOCKHAND_HOST` | Postgres runs on a private `backend` network, never exposed. Set `POSTGRES_PASSWORD`. Optional Authelia middleware label is commented in the compose file. |
| `qbittorrent/` | `SUBDOMAIN` etc. | WebUI router already has `middlewares=authelia@docker`. BT ports are published on the host — forward them on your router. |
| `rutorrent-user1/` | `RUTORRENT_HOST`, `WEBDAV_HOST`, `XMLRPC_HOST` | Instance 1: BT 50000/tcp + 6881/udp, internal 8000/8080/9000. Optional `geoip` profile service. |
| `rutorrent-user2/` | same var names | Instance 2, fully renamed and port-shifted: BT **51000/tcp + 6882/udp**, internal 8100/8180/9100. Forward both port pairs separately. |
| `sftpgo/` | `SFTPGO_HOST` | SFTP :2022, WebDAV :8081, web admin via Traefik. Host dirs must be owned by uid/gid 1000 or sqlite crash-loops. Never name compose-only vars `SFTPGO_*` — they'd hijack app config (host paths use `SFTPGO_HOST_*`). |
| `filezilla/` | `FILEZILLA_HOST` | Browser-streamed desktop GUI. **The in-GUI terminal has passwordless sudo** — put `authelia@docker` in front (strongly recommended) and/or set `CUSTOM_USER`/`PASSWORD`. |

### Running the two ruTorrent instances side by side

They are fully decoupled: separate compose project names, container names
(`rutorrent-user1` / `rutorrent-user2`), router names, hostnames, BT ports,
and volume paths (relative to each stack's own directory). Just make sure the
`.env` of each points `*_HOST` at real DNS records and forward both BT port
pairs (50000/tcp, 6881/udp **and** 51000/tcp, 6882/udp) on your router.

---

## Conventions used across all stacks

- **No host-published web ports.** Web UIs use `expose` and are reachable only
  through Traefik on the `proxy` network. Only peer/transfer traffic
  (BitTorrent, SFTP) is published on the host.
- **All config via `.env`.** Each stack reads `.env` through `env_file`; the
  compose files carry sane `${VAR:-default}` fallbacks. Never commit your
  real `.env` — only `.env.example` lives here.
- **Unique router/service names.** Traefik IDs must be unique across your
  whole Docker host; the rutorrent pair demonstrates the rename pattern.
- **Hardening defaults.** `no-new-privileges`, json-file log rotation
  (10m × 3), internal `backend` networks for databases.
- **Fail fast.** Required vars use `${VAR:?message}` so compose refuses to
  start with a clear error instead of misrouting.

## Typical bring-up order

```bash
docker network create proxy
cd traefik   && docker compose up -d   # certs + routing
cd ../authelia && docker compose up -d # SSO portal (before app routers)
cd ../dockhand && docker compose up -d # then any app stacks
```

## Removing a stack

```bash
cd <stack-dir>
docker compose down          # add -v to delete its named volumes too
```

The shared `proxy` network stays until you `docker network rm proxy`.
