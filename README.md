# Homelab

Docker Compose stack for a personal media and ebook homelab, tuned for **[Bazzite](https://bazzite.gg/)** (Fedora Atomic / immutable desktop).

## Directory layout

```
/home/user/homelab/              ← this repo (HOMELAB_DIR)
├── docker-compose.yml
├── caddy/Caddyfile              ← reverse proxy config (tracked in git)
├── homer/config.yml             ← dashboard links (tracked in git)
└── .env                         ← local paths and secrets (not in git)

/home/user/homelab-config/       ← service configs (CONFIG_DIR, not in git)
/home/user/data/                 ← media, torrents, books (DATA_DIR, not in git)
```

## Bazzite notes

- Volume mounts use the `:z` SELinux flag, which is required on Fedora-based systems like Bazzite when bind-mounting host paths into containers.
- Keep the repo under your home directory (e.g. `/home/user/homelab`). Bazzite's immutable root filesystem is not meant for mutable app data — use `HOMELAB_DIR`, `CONFIG_DIR`, and `DATA_DIR` under `/home/user` instead.
- Install [Docker](https://docs.docker.com/engine/install/) or use Podman with `podman-compose` / `docker compose` compatibility. Either works; adjust commands if you prefer rootless Podman.
- Set `PUID` and `PGID` to your user (`id -u` / `id -g`). On Bazzite the default user is typically `1000`.

## Setup

1. Clone the repo

2. Create your environment file:

   ```bash
   cp .env.example .env
   ```

3. Edit `.env` — at minimum set `HOMELAB_DIR`, `CONFIG_DIR`, and `DATA_DIR` to match your home directory layout.

4. Create the data directories:

   ```bash
   mkdir -p ~/homelab-config
   mkdir -p ~/data/{media,torrents,books/ingest,books/library}
   ```

5. Start the stack:

   ```bash
   docker compose up -d
   ```

6. Open the dashboard at **https://homelab.mtttgl.dev** (via Caddy) or **http://localhost:8081** directly.

## Caddy reverse proxy

[Caddy](https://caddyserver.com/) terminates HTTPS and routes subdomains to each service. Config lives in `caddy/Caddyfile`.

| URL | Service |
|---|---|
| https://homelab.mtttgl.dev | Homer dashboard |
| https://jellyfin.mtttgl.dev | Jellyfin |
| https://seerr.mtttgl.dev | Seerr |
| https://sonarr.mtttgl.dev | Sonarr |
| https://radarr.mtttgl.dev | Radarr |
| https://bazarr.mtttgl.dev | Bazarr |
| https://prowlarr.mtttgl.dev | Prowlarr |
| https://qbittorrent.mtttgl.dev | qBittorrent |
| https://cwa.mtttgl.dev | Calibre-Web-Automated |
| https://shelfmark.mtttgl.dev | Shelfmark |

### DNS (Tailscale)

Point each subdomain (or a wildcard `*.mtttgl.dev`) to your server's **Tailscale IP** in the [Vercel DNS dashboard](https://vercel.com/dashboard/domains).

### TLS (Let's Encrypt via Vercel DNS)

Caddy is built with the [Vercel DNS plugin](https://github.com/caddy-dns/vercel) so certificates are issued with **DNS-01**. That works when your domain points at a **Tailscale IP** — you do not need port 443 open on the public internet.

1. In Vercel, open **Account Settings → Tokens** and create an API token.
2. Ensure `mtttgl.dev` DNS is managed in Vercel (Domains → your domain → DNS Records).
3. In `.env`, set:
   - `ACME_EMAIL` — your email for Let's Encrypt
   - `VERCEL_API_TOKEN` — the token from step 1
4. Rebuild and restart Caddy:
   ```bash
   podman compose build caddy && podman compose up -d caddy
   ```

Caddy will obtain trusted certificates for all hostnames in the Caddyfile. HTTP is redirected to HTTPS automatically.

Note: Vercel limits DNS API calls to **100/hour**. First startup requests certs for ~10 hostnames; renewals are infrequent.

If you prefer self-signed certs (no API token), use `tls internal` in the `(common)` block and the stock `caddy:2-alpine` image instead of the custom build.

### After first start

Configure each app to know its public URL (required for redirects and auth):

- **Jellyfin:** Dashboard → Networking → Known Proxies: add `caddy`; set Base URL if needed
- **Seerr:** Settings → General → Application URL: `https://seerr.mtttgl.dev`
- **Sonarr / Radarr / Prowlarr:** Settings → General → URL Base: leave empty; set host URL in application settings if prompted

FlareSolverr is intentionally not exposed through Caddy (internal use only).

## Default credentials

### qBittorrent

The [LinuxServer image](https://docs.linuxserver.io/images/docker-qbittorrent/) does **not** use a fixed password. On first start:

- **Username:** `admin`
- **Password:** a random temporary value printed in the container logs

```bash
docker logs qbittorrent 2>&1 | grep -A1 "temporary password"
```

Look for a line like: `A temporary password is provided for this session: …`

Set your own password under **Settings → Web UI** after logging in. If you skip this, a new random password is generated on every container restart.

### Calibre-Web-Automated

Default login on first start: `admin` / `admin123` — change this immediately in the web UI.

Optional: set `HARDCOVER_TOKEN` in `.env` for Hardcover metadata. After CWA is running, you can uncomment the `app.db` mount in `docker-compose.yml` to let Shelfmark reuse CWA users for login.
