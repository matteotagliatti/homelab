# Homelab

Docker Compose stack for a personal media and ebook homelab, tuned for **[Bazzite](https://bazzite.gg/)** (Fedora Atomic / immutable desktop).

## Directory layout

```
/home/user/homelab/              ← this repo (HOMELAB_DIR)
├── docker-compose.yml
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

6. Open the dashboard at **http://localhost:8081** and edit `homer/config.yml` to update service links.

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
