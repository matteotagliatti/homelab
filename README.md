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
- **Rootless Podman — required for `https://homelab.mtttgl.dev` (ports 80/443):** run this **once** on the server before starting Caddy:
  ```bash
  sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
  echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-rootless-privileged-ports.conf
  sysctl net.ipv4.ip_unprivileged_port_start   # must print 80
  ```
  Without this, rootless Podman cannot bind port 80 and Caddy will not start. After sysctl, use `CADDY_HTTP_PORT=80` and `CADDY_HTTPS_PORT=443` in `.env` (defaults in `.env.example`).
- Set `PUID` and `PGID` in `.env`. Use your user IDs (`id -u` / `id -g`) for most services. On **rootless Podman**, LinuxServer images often need `PUID=0` and `PGID=0` so container root maps to your host user (see file permissions in `CONFIG_DIR` / `DATA_DIR`).

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

5. **Rootless Podman only:** allow binding ports 80/443 (skip if using Docker, or if already done):

   ```bash
   sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
   echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-rootless-privileged-ports.conf
   ```

6. Start the stack:

   ```bash
   podman compose up -d
   ```

7. Open the dashboard at **https://homelab.mtttgl.dev** (via Caddy) or **http://localhost:8081** directly.

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

The build pins **Caddy 2.9.1** because the Vercel libdns provider has not been updated for libdns v1 (required by Caddy 2.10+).

1. In Vercel, open **Account Settings → Tokens** and create an API token.
2. Ensure `mtttgl.dev` DNS is managed in Vercel (Domains → your domain → DNS Records).
3. In `.env`, set:
   - `ACME_EMAIL` — your email for Let's Encrypt
   - `VERCEL_API_TOKEN` — the token from step 1
4. Rebuild and restart Caddy:
   ```bash
   podman compose build caddy && podman compose up -d caddy
   ```

On **Fedora / Bazzite**, Podman may try to pull short image names from `registry.fedoraproject.org`. The Caddy `Dockerfile` uses fully qualified `docker.io/library/caddy` references to avoid that.

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
